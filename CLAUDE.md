# CLAUDE.md — Frankie

Persistent context for this repo. Read this before every session. Full detail lives in `docs/PRD.md` (reference by section; do not implement the whole PRD at once — build against `BUILD_PLAN_PHASE1.md`).

## What Frankie is
SMS-based (Telnyx) family weekend-planning **agent**, backed by the SMG newsletter. Thursday server-side job generates a per-user weekend plan → delivered as email schedule + SMS. 24/7 interactive agent the rest of the week. Flares = low-pressure social broadcasts.

## The six non-negotiables
1. **Agent, not a parser.** Frankie reasons over tools. Never `if message.contains(...)`. (This was v1's fatal flaw.)
2. **Email-first generation.** Plan is generated ONCE server-side, stored, then email + SMS read from it. Mid-week SMS remixes the stored plan, never regenerates.
3. **Postgres is the single source of truth.** Never JSON files for state.
4. **Quality floor = the product.** Every plan respects the user's zone (QUAL-1) and price_pref (QUAL-2). Hard gates.
5. **Consent is legal.** Only text opted-in users. Non-users only via Invite. Wrong = Telnyx number killed.
6. **All time is America/Los_Angeles.** Store UTC, convert at the edge.

## Stack (decided — do not propose alternatives)
- Hosting: managed platform (Render/Railway/Fly) · DB: managed Postgres
- Billing: Stripe Checkout + Customer Portal (no in-house card UI)
- SMS: Telnyx + 10DLC · Newsletter: Beehiiv (existing, don't touch) · Transactional email: Postmark/Resend
- Calendar: Google OAuth **read/write** · Places: Google, cached 24h
- LLM: two-tier routing (cheap for lookups, reasoning for planning) · Jobs: managed queue, idempotent

## Working rules (how I want you to operate)
- **Plan first, build second.** No implementation until ~95% confident in the approach. State the plan, get confirmation, then build.
- **Hold the priority stack.** P0 before any scope expansion. Don't gold-plate.
- **One vertical slice at a time** per `BUILD_PLAN_PHASE1.md`. Each slice ships end-to-end and has acceptance criteria. Don't start the next slice until the current one passes.
- **Infra discipline:** before anything touching a live server, confirm exact deploy path, service name, and running user. Never assume. Never write a deploy script as a local file and assume it ran — SSH and execute directly, then print a summary.
- **No tool-research detours.** Don't suggest evaluating new tools mid-task. If you catch yourself reaching for one, flag it and stop.
- **Acceptance criteria are the definition of done.** A slice isn't done because code exists — it's done when its criteria pass. Say which criteria you verified.
- **Snapshots not live FKs** for `plan.blocks.venue_ref` — a stored plan must survive the event being deleted.

## Repo conventions
- `docs/PRD.md` — source of truth, reference only
- `BUILD_PLAN_PHASE1.md` — the actual marching orders
- Secrets in env vars, never committed. OAuth + Stripe tokens encrypted at rest.
- Migrations for every schema change; never hand-edit prod schema.

## Out of scope until told otherwise
Flares, Groups, Invites, referral loop, premium tier, win-back/marketing sequences. These are real and in the PRD, but they are NOT Phase 1. Phase 1 = the core loop that must be excellent before the waitlist sees Frankie.
