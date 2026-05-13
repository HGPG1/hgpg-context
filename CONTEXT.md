# HGPG Context (Brain)

Last updated: 2026-05-12 (end-of-day reconciliation pass)

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## What is active right now

- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, ongoing Don feedback batches, see `projects/transaction-manager.md`
- **TC Concierge** (tc.homegrownpropertygroup.com) - email intake, Don running real deals, see `projects/tc-concierge.md`
- **CMA Engine** (cma.homegrownpropertygroup.com) - awaiting Taylor stress test, see `projects/cma-engine.md`
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - live, see `projects/listing-report-portal.md`
- **Brain App** (brain.homegrownpropertygroup.com) - live, see `projects/brain-app.md`
- **FUB AI Agent** (embedded in TM at `/agent`) - operational, `agent_enabled=false`, decision pending on flip strategy, see `projects/fub-ai-agent.md`
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com) - shipped + ad-ready, FUB Automation 2.0 pending, see `projects/sellers-guide.md`
- **Buyers Guide** (buyersguide.homegrownpropertygroup.com) - **Sessions 1+2 of Manus migration shipped 2026-05-12**. Backend infrastructure + Calculator + Quiz + Exit Intent + Bonus Unlock all live. Sessions 3+4 queued. See `projects/buyers-guide.md` and `projects/buyers-guide-manus-migration.md`.
- **New Construction site** (newconstruction.homegrownpropertygroup.com) - phone capture tiered downgrade shipped 2026-05-12 (PR #1, merge `8bca9ac`). SMS speed-to-lead automation parked. See `projects/new-construction-phone-capture.md`.
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Recently completed

- **Buyers Guide Manus migration Session 2** (2026-05-12) - Calculator + Quiz + Exit Intent + Bonus Unlock shipped end-to-end
- **Buyers Guide Manus migration Session 1** (2026-05-12) - Supabase backend (bg_contacts, bg_activities, bg_quiz_results, bg_agents), server helpers, FRED/FUB/Storage health endpoints
- **New Construction phone capture** (2026-05-12) - tiered required/optional across 4 forms, E.164 normalization, per-form helper copy. PR #1 merged.
- **Buyers Guide instrumentation** (2026-05-12) - Meta Pixel + CAPI made live (was inert since April), NeverBounce shipped from scratch. Commit `5c233ee`, PR merged.
- **Manus full source extraction** (2026-05-12) - 220-file snapshot in `HGPG1/charlotte-buyers-guide-manus-export`. Live Manus DB confirmed effectively empty via tRPC probe.
- **transaction-pdfs bucket flipped to private** (2026-05-11) - bucket private since 2026-05-09, code on main via PR #7, migration file backfilled to repo (`2eb9794`)
- **NC office routing in ReZEN builder verified wired** (2026-05-11)
- **Sherlock 403 on Transaction Manager resolved** (2026-05-09)
- **Listing Report Portal GitHub auth blocker resolved** (2026-05-06)
- **Exposed GitHub PAT closed out** (2026-05-11) - superseded by fine-grained brain-app PAT
- **CMA Engine MLS Grid auto-pull confirmed mature** (2026-05-09)
- **Brain App MVP + Phase 1.5 + Write API shipped** (2026-05-06 → 2026-05-08)
- **Resend SMTP wired into HGPG Core Supabase** (30/hr vs 2/hr default)
- **IDX Broker migration** (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- **Main site SEO push** (mobile PageSpeed 96/100)
- **Sellers guide rebrand to brand colors + ad instrumentation**

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- **🟡 Quiz lifestyle-key mismatch verification needed** - `bg_quiz_results` table has 0 rows as of 2026-05-12 end-of-day despite Session 2 smoke test putting 3 rows in `bg_contacts` + `bg_activities`. Could mean the bug paused at commit `879d263` was never fully resolved before Session 2 was marked shipped, OR quiz simply wasn't smoke-tested. Confirm BEFORE Session 3 modifies Quiz to add advisor mode.
- **Buyers Guide Manus migration Sessions 3+4** - queued, prompts staged in `projects/buyers-guide-manus-migration.md`
- **SMS speed-to-lead automation** for New Construction - parked decision (gate on SMS consent vs fire on Builder Intro). Recommendation: gate on consent for TCPA defensibility.
- **`META_CAPI_TEST_EVENT_CODE` cleanup** on Buyers Guide once browser+server dedup confirmed in Meta Test Events
- **`$395 fee toggle`** structural build parked (TM, commit `b9fa0deb`)
- **TC Concierge absorption into TM** deferred (current architecture stable)
- **Listing Report Portal DB pruning** parked (~15GB, 8-10GB reclaim estimated)
- **FUB AI Agent flip strategy decision** pending (`agent_enabled=true` vs sustained manual-approve)
- **FUB AI Agent scoring sweep** on 4,340 unscored eligible leads (deferred)
- **Sellers Guide FUB Automation 2.0** on `sellers-guide-2026` tag not yet built (blocks ad-spend scaling)
- **`~/Downloads/fix-admin-page.sh`** may have unpushed Listing Report Portal UI changes
- **.net Google Workspace migration to .com** (do not proactively remind)
- **Take Manus app down** (no 301) once Buyers Guide migration Sessions 3+4 complete

## CONTEXT.md drift root cause + fix

2026-05-12 reconciliation revealed this file had been drifting again — top-line timestamp got bumped but body lagged by days. Three structural causes:

1. **Two-layer brain, one-layer attention.** Each Claude Code session works inside a specific `projects/*.md` file. CONTEXT.md is the index — nobody owns it during a focused session. Gets brushed at session end if it gets brushed at all.
2. **Different agents have different mental models of what CONTEXT.md should contain.** Some update "active." Some update "completed." Nobody updates all three sections together because there's no checklist.
3. **No canonical schema.** CONTEXT.md isn't generated, just hand-maintained.

**Phase 2 fix (deferred):** auto-generate the "What is active" and "Recently completed" sections from structured headers in each `projects/*.md`. Each project gets a YAML frontmatter or fenced JSON block with `status`, `lastShipped`, `summary`. A pre-commit hook or scheduled brain-app cron rebuilds the index sections from project files. Lightweight alternative: a `CHECKLIST.md` at brain root listing what to touch in CONTEXT.md at session end, referenced from session kickoff prompts.

## Where to look next

| Topic | File |
|---|---|
| Brand colors, fonts, voice | `brand.md` |
| Team, FUB IDs, contractors | `team.md` |
| GitHub/Vercel/Supabase/DNS | `infrastructure.md` |
| SC compliance, Lando Law | `compliance.md` |
| Meta Ads, lead re-engagement | `marketing.md` |
| Session protocol, formatting, iMessage | `operations.md` |
| Specific project state | `projects/<name>.md` |
