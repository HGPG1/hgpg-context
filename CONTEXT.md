# HGPG Context (Brain)

Last updated: 2026-05-20

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## What's active right now

- **CMA Engine** (cma.homegrownpropertygroup.com) - MLS Grid auto-pull comps live in production. See `projects/cma-engine.md`
- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, see `projects/transaction-manager.md`. Remaining: addendum block injection, per-party messaging capability tracking, calendar gate backfill. Agent onboarding (Ashley/Taylor/Brenda) gated on first full deal through system.
- **TC Concierge** - embedded at `/intake` in TM, live with Don running real deals. Real-life classifications/extractions worth reviewing in next session. Standalone `concierge.homegrownpropertygroup.com` (503) is legacy and slated for decommission.
- **Sellers Guide Meta ads** (sellersguide.homegrownpropertygroup.com) - campaign shipped 2026-05-15 (Friday). FUB Automation 2.0 build likely ready, validation gates on first real ad lead. See `projects/sellers-guide.md`
- **Charlotte New Construction** (newconstruction.homegrownpropertygroup.com) - Variant D + Variant E both live; Variant E launched 2026-05-18 with full scoped package, day 3-4 read window opens 2026-05-21. See `projects/charlotte-new-construction.md`
- **IDXRE-2026-04 B2** - Automations 336 + 337 built and wired to templates 1166 + 1167 (verified live 2026-05-19). KTS contamination shut down 2026-05-18 and pause held (verified by indirect signal). Launch gates remaining: UI verification of audience filters + exclusion, pick launch date, enable. See `projects/idxre-2026-04.md`
- **FUB AI Agent** - operational + monitored. Week 1 ramp at daily_send_cap=10 with manual approve. Humming. See `projects/fub-ai-agent.md`
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - GitHub auth resolved on Mac mini; ready to unblock. See `projects/listing-report-portal.md`
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Recently completed

- KTS legacy automation contamination shut down: 6,625 expired leads bulk-applied to `bulk` Automation 2.0, 3 active KTS automations paused at automation level (2026-05-18)
- IDXRE B2 build verified (2026-05-19): Automations 336 (Sellers) + 337 (Buyers) live and wired to templates 1166 + 1167. KTS pause held (all B1 tier counts intact, no Hot-lead activity post-2026-05-17). Launch gates: UI audience-filter verification + enable.
- B2 templates created via FUB API: 1166 (Sellers - Just Checking In) and 1167 (Buyers - Market Shifted), targeting ~693 + ~304 leads (2026-05-18)
- Sellers Guide Meta ads campaign shipped (2026-05-15)
- MLS Grid token wired - CMA Engine auto-pull comps live in production
- Brain App MVP shipped at brain.homegrownpropertygroup.com (2026-05-06)
- Brain App `/api/external/commit` endpoint added for app-code commits to any HGPG1 repo (2026-05-14)
- Brain App `/api/files` reads now require bearer auth (2026-05-15)
- HGPG Listing Reports + MLS Resend SMTP wired (2026-05-15)
- IDXRE-2026-04 B1 campaign scored: 1,289 leads tiered Hot/Warm/Cool/Cold/Dead; 1,284 segment-tagged Seller/Buyer/Unknown; 135 Dead moved to pond 17 IDXRE-Triage (2026-05-17)
- Sherlock 403 on TM resolved
- GitHub auth resolved on Mac mini
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers Guide Session 3 shipped (agent slug routes /brian /brenda /ashley /taylor + AdvisorMode + assignedAgentSlug plumbed end-to-end) — production 2026-05-19, real-traffic leads with named agents captured same day. See `projects/buyers-guide.md`
- New Construction Variant E launched (2026-05-18) — full scoped package: brief PDF + three creatives (1080x1080, 1080x1350, 1080x1920) using IMG_5675, $10K+ closing-cost dollar anchor, "Learn More" CTA, `utm_content=variant-e-10k`. Day 3-4 read window 2026-05-21 → 2026-05-22 against Variant C's $4.63 CPL benchmark. See `marketing.md` + `projects/charlotte-new-construction.md`
- Buyers guide migration from Manus to React + Vite (Sessions 1+2 live 2026-05-12; Session 3 live 2026-05-19; Session 4 optional, queued)
- Sellers guide rebrand to brand colors

## Parked (not active, do not pick up without explicit trigger)

- **Buyer Alerts** (Team Dashboard `/buyers`) — LoopMessage patch DRAFTED, not committed. Wait for clean session to pair items 1+2+3
- **TM $395 fee toggle** — build spec parked, awaiting greenlight
- **DocuSign migration off zipForms** — workflow scoped, target 2026-06-13; re-evaluate zipForms pain before starting
- **NC "For Builders" footer link** — low priority, do next time touching the NC site
- **Buyers Guide Session 4** — Interactive Map + Market Pulse + bonus PDFs (~1.5 hrs, optional, not yet built). Manus app takedown follows S4.

## On the docket for next session

- **Phone Capture Rate Review** — run window opened 2026-05-19, query plan in `projects/new-construction-phone-capture-rate-review.md`
- **Incentives Funnel Phase 1 → Phase 2 calendar checkpoint** — ~2026-06-02, review whether Variant A/B/C data is mature enough to design Phase 2
- **TC Concierge real-life extraction review** — Brian to share live deals for classification/extraction quality check

## On the recheck list (lower urgency)

- **Incentives Funnel Phase 2 design** (separate from the calendar checkpoint above) — possible directions logged but not committed
- **NC Quiz Scoring calibration** — two of five quiz inputs (commute, firstTimeBuyer) are dead weight for scoring; either wire in or strip to reduce friction. Deferred until first 50 qualified leads
- **CRM brain stub buildout** — `projects/crm.md` was created 2026-05-17 as a 🟡 stub; needs active campaign state and Recent FUB Work entries appended as sessions ship

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- B2 IDXRE launch gates: UI verification of audience filters + Hot/Warm exclusion on Automations 336/337, date-stamp tag, enable automations. KTS pause verified 2026-05-19.
- IDXRE Hot tier outreach PARKED until contamination-aware re-scoring possible
- Stack consolidation parked ~2-3 weeks: kill dead concierge repo, migrate `deals` → `transactions`, fix Title Case vs snake_case status mismatch
- .net Google Workspace migration to .com (do not proactively remind)
- GitHub PAT exposed in chat history needs rotation

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
| Cluster C TM RLS hardening (Claude Code brief) | `projects/cluster-c-path-3-brief.md` |
