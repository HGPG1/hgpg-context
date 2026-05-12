# HGPG Context (Brain)

Last updated: 2026-05-12

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
- **CMA Engine** (cma.homegrownpropertygroup.com) - production-ready, 21-PR sweep complete (engine math + autosave + mobile + addresses + narrative cache + back-links + comp persistence), awaiting real-world agent reps, see `projects/cma-engine.md`
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - live, see `projects/listing-report-portal.md`
- **Brain App** (brain.homegrownpropertygroup.com) - live, see `projects/brain-app.md`
- **FUB AI Agent** (embedded in TM at `/agent`) - operational, `agent_enabled=false`, decision pending on flip strategy, see `projects/fub-ai-agent.md`
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com) - shipped + ad-ready, FUB Automation 2.0 pending, see `projects/sellers-guide.md`
- **Buyers Guide** (buyersguide.homegrownpropertygroup.com) - live, but **major Manus migration workstream active** — Vercel rebuild missing 8 high-value features from original (lead scoring, calculator→FUB tags, exit-intent + PDF gen, bonus tracking, /:agent vanity pages, /admin, /agent-dashboard, advisor mode). See `projects/buyers-guide.md`.
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Recently completed

- Buyers Guide Pixel + CAPI + NeverBounce shipped (2026-05-12). PR merged. Env vars + redeploy + verification pending.
- Manus AI agent extraction Rounds 1 + 2 (2026-05-12) — repo `HGPG1/homegrown-buyer-guide` confirmed, full file tree + Drizzle schema in hand. Round 3 (full source) + Round 4 (DB CSV export) queued.
- transaction-pdfs bucket flipped to private (fully aligned 2026-05-11)
- NC office routing in ReZEN builder verified wired (2026-05-11)
- Exposed GitHub PAT closed out (2026-05-11)
- CMA Engine MLS Grid auto-pull confirmed mature (2026-05-09)
- Sherlock 403 on Transaction Manager resolved (2026-05-11)
- Listing Report Portal GitHub auth blocker resolved (2026-05-06)
- Brain App MVP shipped (2026-05-06), Phase 1.5 polish + Write API (2026-05-08)
- Resend SMTP wired into HGPG Core Supabase (30/hr vs 2/hr default)
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Sellers guide rebrand to brand colors + ad instrumentation

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- **Buyers Guide Manus extraction Rounds 3 + 4** — Path B (full source dump) and Path C (DB CSV export). High priority; extract BEFORE telegraphing departure to Manus.
- **Buyers Guide Pixel + CAPI verification** — env vars need provisioning on Vercel; redeploy and verify dedup in Meta Test Events.
- **Buyers Guide gap-port to Vercel** — multi-session workstream once Manus export is in hand. Phase 2 (lead scoring, calc→FUB, PDF gen, exit intent, bonus tracking) is highest priority.
- $395 fee toggle structural build parked (TM)
- TC Concierge absorption into TM deferred (current architecture stable)
- Listing Report Portal DB pruning parked (~15GB, 8-10GB reclaim estimated)
- FUB AI Agent flip strategy decision pending (`agent_enabled=true` vs sustained manual-approve)
- FUB AI Agent scoring sweep on 4,340 unscored eligible leads (deferred)
- Sellers Guide: FUB Automation 2.0 on `sellers-guide-2026` tag not yet built (blocks ad-spend scaling)
- .net Google Workspace migration to .com (do not proactively remind)

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
