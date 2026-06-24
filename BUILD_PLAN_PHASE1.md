# Frankie — Phase 1 Build Plan

**Goal of Phase 1:** the smallest Frankie that is *excellent* — the core loop a warm waitlist name would judge. If this isn't good, nothing else matters. Everything else (Flares, Groups, premium, marketing sequences) waits.

**Phase 1 done =** a real user can onboard on the web, receive a genuinely good (QUAL-passing) weekend plan by email + SMS on Thursday, text Frankie mid-week for an alternative and get a reasoned (not parsed) reply, and one-tap a plan item onto their calendar.

**How to use this file:** each slice is one scoped Claude Code session. Do them in order. Don't start a slice until the previous one's acceptance criteria pass. Each slice is a vertical cut — it should work end-to-end, not be a half-layer.

---

## Slice 0 — Foundations
**Build:** clean repo, managed Postgres, the full schema from `docs/PRD.md §2` as migrations, managed hosting with push-to-deploy, env-var secrets. Create (don't yet wire) Stripe, Telnyx, transactional-email accounts. **Start 10DLC registration now** — it has lead time and blocks launch, not the build.

**Acceptance criteria:**
- [ ] All entities (User, Plan, Venue, Group, Flare, Invite) exist as migrations and apply cleanly to a fresh DB.
- [ ] App deploys to the managed platform via push; health check passes.
- [ ] 10DLC brand/campaign registration submitted.
- [ ] No secrets in the repo.

---

## Slice 1 — Onboarding writes a real user
**Build:** the web onboarding flow from `§9`. Captures phone, email, interests, home zone, **price preference**, calendar connect (Google OAuth **read/write**) or skip-to-free-windows, optional shared emails, SMS consent (store `sms_opt_in_at`), and Stripe Checkout. A completed signup is a real row in `users` with `account_status = trial`.

**Acceptance criteria:**
- [ ] Completing onboarding creates a User row with zone, `price_pref`, and consent timestamp set.
- [ ] Calendar connect stores an encrypted read/write token; skip stores `free_windows`.
- [ ] Stripe Checkout collects card-on-file; first charge deferred 30 days (no immediate charge).
- [ ] STOP/HELP handlers exist and a STOP marks the user opted-out immediately.

---

## Slice 2 — Events importer + venue data
**Build:** the Google-Sheet → Postgres importer from `§3 / EVT`. Owner-triggered, validates per-row, reports errors (doesn't fail silently), stamps `last_updated`. Seed the evergreen venue list and the curated at-home fallback list.

**Acceptance criteria:**
- [ ] A valid sheet upserts into `venues` (kind=event); a row missing a required field is rejected by name without blocking valid rows.
- [ ] Evergreen + at-home lists are loaded and queryable by zone + price.
- [ ] `last_updated` is stamped on import.

---

## Slice 3 — Generation worker (the heart) + quality eval
**Build:** the idempotent Thursday generation job from `§1.2 / §3`. Reads a user, finds calendar gaps (or uses free_windows), queries events + evergreen + Places (24h cache), assembles a Plan respecting **QUAL-1 (zone) and QUAL-2 (price)**, one anchor per block, snapshots `venue_ref`. Includes the stale-sheet fallback (EVT-6/7) and the empty-calendar fallback (GEN-5/6). **Ship the QUAL eval (QUAL-4)** — fixed test personas, auto-checked for zone/price violations.

**Acceptance criteria:**
- [ ] For every test persona, zero suggestions violate zone or price. Eval runs green.
- [ ] A fully-booked calendar still yields ≥1 suggestion with `fallback_used = true`.
- [ ] A stale events sheet never surfaces a past-dated event; "quiet week" framing is used.
- [ ] Job is idempotent: re-running for the same user/weekend doesn't duplicate or double-send.
- [ ] `venue_ref` is a snapshot — deleting the source venue doesn't break an existing Plan.

---

## Slice 4 — Delivery (email + SMS)
**Build:** Thursday delivery from `§1.4`. Transactional sender renders the per-user schedule email (+ shared recipients); Telnyx sends the SMS summary built from the *same* stored Plan. Both carry the "double-check before you leave" line. Scheduled at **4pm PT**.

**Acceptance criteria:**
- [ ] One generated Plan produces a matching email and SMS that never disagree.
- [ ] Shared-email recipients receive the schedule; they have no account and can't text back.
- [ ] Send fires at 4pm PT (not UTC midnight).
- [ ] Email has one-click unsubscribe + physical address.

---

## Slice 5 — Interactive agent (NOT a parser)
**Build:** the 24/7 agent from `§4` as an **LLM with tool-calling**. Tools: read stored plan, query events/evergreen/at-home, query Places, read calendar gaps, **add-to-calendar (one-tap, user-initiated)**, read/update prefs. Two-tier model routing (RL-1). Per-week cap reset Monday 00:00 PT (RL-2/3) + hard auto-throttle ceiling (RL-6). All six failure rows from `§4` defined and tested. Scope guardrail: family planning only, redirect off-topic.

**Acceptance criteria:**
- [ ] "Anything indoor, it's raining?" returns a reasoned, zone+price-respecting alternative — verified it is NOT keyword-matched.
- [ ] Simple lookups route to the cheap model; planning routes to the reasoning model.
- [ ] Every failure row (Places down, no events, expired token, not-understood, off-topic, tool error) has a defined, tested response — no raw errors, no silence.
- [ ] Hitting the weekly cap returns the on-brand message; the hard ceiling auto-throttles without owner action.

---

## Slice 6 — One-tap calendar slotting
**Build:** CAL-1/2/3. A tap on a plan item creates exactly one Google Calendar event (title, time, venue, address) using the write scope. Never auto-writes.

**Acceptance criteria:**
- [ ] Tapping "add" creates exactly one correct event and writes nothing un-confirmed.
- [ ] No code path auto-writes to the calendar.

---

## Phase 1 exit check (all must be true before a waitlist name sees Frankie)
- [ ] A real person can onboard, get a Thursday plan by email + SMS, text for an alternative, and slot an item — end to end, no manual steps.
- [ ] The QUAL eval is green; spot-checked real plans actually feel good (owner judgment).
- [ ] STOP works; the Telnyx number is 10DLC-registered.
- [ ] Nothing in Phase 1 depends on Flares/Groups/premium existing.

**Then, and only then:** Phase 2 (agent depth), Phase 3 (Flares/Groups/Invites), Phase 4 (billing polish), and the marketing sequences — in that order.

---

## Guardrail for the owner
The danger isn't building too little — it's treating the whole PRD as the build. Hold the line: Phase 1 is six slices. Resist adding Flare/Group work "while you're in there." A great six-slice core in front of one real user beats a half-built everything.
