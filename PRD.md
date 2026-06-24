# Frankie — Product Requirements (Source of Truth)

> **What this file is:** the canonical reference for *what* Frankie is and *why* each decision was made.
> **What this file is NOT:** a build instruction. Do not implement "the whole PRD." Build against `BUILD_PLAN_PHASE1.md`, one slice at a time, and use this file to look up detail by section.

Frankie is a conversational, **SMS-based** family weekend-planning agent (delivered over **Telnyx**), backed by the *Shitty Mom Guide* (SMG) weekly newsletter. Each Thursday a server-side job generates a personalized weekend plan per user — from calendar gaps, home zone, interests, and **price preference** — and delivers it as an **email schedule + an SMS**. Beyond that, Frankie is a **24/7 interactive agent** (NOT a keyword parser). A **Flare** lets a user broadcast low-pressure intent to friends or a saved Group.

---

## Non-negotiables (read before writing any code)

1. **Frankie is an LLM agent with tools, NOT a keyword/signal parser.** v1 was built as a word-matching parser; that was the core architectural failure. The agent *reasons* over tools. If you find yourself writing `if message.contains("park")`, stop.
2. **Email-first generation is the keystone.** The weekend plan is generated **once, server-side**, as part of the Thursday job and stored as a Plan object. Email and SMS both read from that one stored plan. Mid-week SMS *remixes* the stored plan; it never regenerates from scratch.
3. **Postgres is the single source of truth.** Not JSON files. The web app is the primary interface; Postgres is the datastore.
4. **Quality floor is the product.** Every plan MUST respect the user's location (zone) and price preference. These are hard gates, not style.
5. **Consent boundary is legal, not optional.** Frankie only texts opted-in users. Non-users are reached only via the opt-in Invite. Getting this wrong gets the Telnyx number shut down.
6. **All time logic is America/Los_Angeles (Pacific).** Store UTC, convert at the boundary.

---

## 1. Architecture

### 1.1 Stack (decided — do not re-litigate)

| Concern | Choice | Rationale |
|---|---|---|
| Hosting | Managed app platform (Render/Railway/Fly) | No server babysitting; push-to-deploy |
| Datastore | Managed Postgres | The "flat queryable" need; a data lake is over-engineering |
| Billing | Stripe (Checkout + Customer Portal) | Stripe hosts card form + cancellation UI; no PCI surface |
| SMS | Telnyx + 10DLC registration | Already chosen; registration has lead time — start day one |
| Newsletter | Beehiiv (existing) | Owns the free SMG newsletter + acquisition funnel. Do NOT move it. |
| Transactional email | Postmark or Resend | Per-user schedule + lifecycle mail. Beehiiv can't render per-recipient DB-driven email. |
| Calendar | Google Calendar API (OAuth, **read/write**) | Read = gap detection; write = one-tap slotting |
| Places | Google Places + Maps, cached 24h | Cache aggressively — cost + rate limits |
| LLM | Two-tier routing (cheap + reasoning) | Cost control is structural |
| Jobs | Managed queue/scheduler | Generation is an idempotent queued job |

### 1.2 Data flow

1. **Thursday trigger** (scheduled, 4pm PT default) enqueues one generation task per active user.
2. **Generation worker** reads the user record, pulls calendar gaps, queries events + evergreen + Places, assembles a plan respecting the QUAL floor.
3. **Stored Plan object** written to Postgres, keyed to user + weekend.
4. **Delivery**: email schedule to user + shared recipients; SMS summary to user.
5. **Mid-week SMS** reads + remixes the stored plan via the agent. Never regenerates.

### 1.3 Time & timezone

- All user-facing time logic in **America/Los_Angeles**. Store UTC.
- "The weekend" = upcoming Sat–Sun Pacific.
- Thursday send: **4:00pm PT** (config-tunable).
- Weekly message cap resets **Monday 00:00 PT** (fixed boundary, not rolling).
- Trial/promo charges fire at a fixed PT hour, never UTC midnight.

### 1.4 Email split

- **Beehiiv**: the free SMG newsletter (broadcast + acquisition funnel). Stays put.
- **Transactional sender**: per-user Thursday schedule + all lifecycle mail (receipts, reminders, invite confirmations).
- Both authenticate the sending domain (SPF/DKIM/DMARC); coordinate DNS.

### 1.5 Acquisition funnel

Freemium-by-product-line: **free = the SMG newsletter** (Beehiiv, top-of-funnel); **paid = Frankie** ($5/mo upgrade). The existing Beehiiv list is the launch audience. Beehiiv list and Frankie user table share an identifier (email) so conversion is measurable.

---

## 2. Data model

UUIDs for ids. `created_at`/`updated_at` on every entity. Types indicative (Postgres).

### User
`id` · `phone` (E.164, unique) · `email` (unique) · `display_name` · `home_zone` (north/central/south) · `home_address?` · `interests` (jsonb) · **`price_pref` (free/$/$$ — hard quality gate)** · `free_windows` (jsonb) · `calendar_connected` (bool) · `google_oauth_token` (encrypted, **read/write** scope) · `tier` (standard/premium) · `account_status` (trial/active/past_due/suspended/cancelled/flagged) · `trial_ends_at` · `shared_emails` (jsonb) · `sms_opt_in_at` · `weekly_msg_count` (int) · `over_cap_strikes` (int)

### Plan
`id` · `user_id` · `weekend_of` (date) · `blocks` (jsonb: array of `{window, anchor_activity, venue_ref, travel_note}` — **`venue_ref` is a SNAPSHOT, not a live FK**; a stored plan must not break if the event is later removed) · `fallback_used` (bool) · `generated_at` · `channel_sent` (jsonb)

### Venue
`id` · `kind` (event/evergreen/at_home) · `title` · `description` · `category` · `zone` · `address?` · `place_id?` · `start_at?` · `end_at?` · `age_fit` (jsonb) · `cost` · `indoor_outdoor` · `status` (active/cancelled)

### Group
`id` · `owner_id` · `name` · `member_user_ids` (jsonb — **Frankie user ids ONLY**; never raw numbers)

### Flare
`id` · `sender_id` · `message` · `recipient_user_ids` (jsonb) · `group_id?` · `delivery_results` (jsonb, per-recipient) · `created_at`

### Invite
`id` · `inviter_id` · `invitee_phone` · `status` (sent/joined/expired) · `grants_free_days` (30)

---

## 3. Weekend plan generation

- **GEN-1** Connected calendar → read events to find open blocks.
- **GEN-2** Declined calendar → use stored `free_windows`; periodically re-ask.
- **GEN-3** ONE anchor activity per open block. No multi-stop itinerary chaining in v1.
- **GEN-4** Travel logic = "near the home zone," not multi-leg routing.
- **GEN-5** Never silent. No gaps → meal idea / at-home / low-effort option.
- **GEN-6** At-home fallback is a CURATED owner list, not LLM-generated.

### Plan quality (the differentiation bar)
- **QUAL-1** Every suggested venue is in/near the user's home zone. Out-of-zone = failure.
- **QUAL-2** Suggestions within the user's `price_pref`. Pushing $$ at a budget user = failure.
- **QUAL-3** Location + price are the v1 floor (necessary), not the ceiling. "Delightful" is post-launch tuning.
- **QUAL-4** Ship a lightweight eval: fixed test personas (zone + price + age-fit) auto-checked that no suggestion violates QUAL-1/2. Run before each release of generation logic.

### One-tap calendar slotting
- **CAL-1** User can add a chosen plan item to Google Calendar in ONE action (title, time, venue, address).
- **CAL-2** ALWAYS user-initiated. Frankie NEVER auto-writes. Auto-slotting is rejected as overbearing.
- **CAL-3** OAuth scope is read/write. Read powers GEN-1; write powers CAL-1. Encrypted at rest.

### Stale-sheet failure plan
- **EVT-5** Importer stamps `last_updated`; generation checks freshness.
- **EVT-6** Stale set → never present old events as current; fall back to evergreen + at-home.
- **EVT-7** Stale week → honest "quiet week" framing. Never silent, never lies about freshness.
- **EVT-8** Owner gets a pre-Thursday alert if the sheet isn't updated for the target weekend.

---

## 4. Interactive agent

**LLM with tool-calling. NOT a keyword parser.**

- **AGT-1** Any-day texting: ask what to do, request alternatives, adjust the plan.
- **AGT-2** Agent is NOT tier-gated; both tiers get 24/7 access. Tiers differ in plan SCOPE.
- **AGT-3** Family "what should we do" planning only; off-topic → friendly redirect.
- **AGT-4** Memory = current weekend's plan + recent session messages. No infinite history.

### Tools
Read stored plan · query events (category/zone) · query evergreen + at-home · query Places (ratings/reviews/travel time) · read calendar gaps · **add plan item to calendar (one-tap, user-initiated)** · read/update saved venues + prefs · compose/send Flare · create/edit Group.

### Failure behavior (each must be defined + tested)
| Failure | Behavior |
|---|---|
| Places down | Fall back to curated data; never block; don't name the outage |
| No events in zone | Offer evergreen + at-home, don't apologize/go empty |
| Calendar token expired | Degrade to `free_windows`; gentle reconnect prompt |
| Not understood | One clarifying question, then redirect to what Frankie does |
| Off-topic | One-line friendly redirect |
| LLM/tool error | Warm generic fallback; log for owner |

---

## 5. Flares & Groups

- **FLR-1** Compose a Flare → send to selected friends or a saved Group.
- **FLR-2** No RSVP, no yes/no capture, no group chat created.
- **FLR-3** Unreachable recipient → tell the SENDER ("couldn't reach her, text her directly"). Per-recipient results stored.
- **GRP-1** Create a named Group: reusable list of Frankie-user friends, owned by creator.
- **GRP-2** Minimal in v1: name + member list. No roles, no chat, no permissions.
- **GRP-3** Members must already be Frankie users (or join via Invite). Never a raw number.
- **FLR-4** Never SMS a non-opted-in number. Frankie-user-only rule = TCPA safety.
- **FLR-5** Reach a non-user only via 30-day free Invite; once joined, Flare-able + Group-addable.

---

## 6. Pricing & billing

| Tier | Price | Scope | Agent | v1? |
|---|---|---|---|---|
| Standard (monthly) | $5/mo | Weekend plan | Full 24/7 | Live |
| Standard (annual) | $50/yr | Weekend plan | Full 24/7 | Live |
| Premium | $12/mo | Full-week plans | Full 24/7 | "Coming soon" (not purchasable) |

- **BIL-3** Signup + payment online via Stripe Checkout. No in-house card UI.
- **BIL-4** Trial: sign up, card on file, first charge after 30 days, pre-charge notice.
- **BIL-5** Summer promo is a **configurable date** (not hard-coded): free until the date, card on file, first charge on that date.
- **Lifecycle:** card fails → 7-day grace, **fully active** → 3 reminders → read-only suspension → cancel at 30 days past due. Cancel = end-of-period. Data retained 90 days then purged.
- **BIL-6/7** One paying user may add a few EMAIL-ONLY recipients (`shared_emails[]`); they have no account and can't text Frankie.

---

## 7. Compliance

- **CMP-1** Register brand + campaign for 10DLC before launch. Start day one (lead time).
- **CMP-2** Paying users opt in to SMS at signup; store `sms_opt_in_at`.
- **CMP-3** Implement STOP/UNSUBSCRIBE (immediate opt-out + confirm) and HELP. Legally mandatory.
- **CMP-4** Frankie texts ONLY consented users. Non-users via Invite only. No exceptions, incl. Groups.
- **CMP-5/6/7** Email: one-click unsubscribe + physical address (CAN-SPAM); SPF/DKIM/DMARC on both Beehiiv and transactional; `shared_emails` can unsubscribe independently.

---

## 8. Rate limiting & cost

- **RL-1** Model routing is HARD: cheap model for lookups, reasoning model only when planning.
- **RL-2** Per-WEEK message bucket, resets Monday 00:00 PT.
- **RL-3** On cap: on-brand "tapped out, back next week," never silent.
- **RL-4** Repeat cap-blowers flagged for owner review + cheeky nudge.
- **RL-6** Flag queue assumes no active triage. The cap is the real protection; add a hard ceiling (~3× weekly cap) that auto-throttles a runaway account without owner action.

---

## 9. Onboarding (web)

1. Phone + email
2. Interests + home zone + **price preference**
3. Calendar connect (read/write) or skip → free windows
4. Optional shared emails
5. SMS consent + payment (Stripe)

---

## 10. Non-functional

- Sized for **low-thousands** of active users, not hyperscale.
- Generation processes all active users within the Thursday window; job is queued + idempotent (crash resumes, no double-send).
- Agent replies feel responsive over SMS (a few seconds); routing keeps lookups fast.
- OAuth + Stripe tokens encrypted; no card data touches our servers.
- Coarse zones now; address-level later without schema rewrite.

### Analytics (log these)
Signup/trial start · **newsletter→Frankie conversion** · plan generated + delivered (per channel) · interactive message (with model tier, for cost) · Flare sent + per-recipient result · Invite sent + joined · cap hit + flag · trial→paid conversion · cancellation.
**Activated user** = received ≥1 plan AND sent ≥1 interactive message or Flare.

---

## Owner inputs still open (safe defaults in config)
1. Summer-promo end date — configurable, TBD
2. Annual price confirm ($50/yr placeholder)
3. Weekly message-cap number (~50/wk to start)
4. Activated-user definition sign-off
