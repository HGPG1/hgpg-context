# HGPG Context (Brain)

Last updated: 2026-05-18

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
- **TC Concierge** - embedded at `/intake` in TM, live with Don running real deals. Standalone `concierge.homegrownpropertygroup.com` (503) is legacy and slated for decommission.
- **Sellers Guide Meta ads** (sellersguide.homegrownpropertygroup.com) - campaign shipped 2026-05-15 (Friday). See `projects/sellers-guide.md`
- **IDXRE-2026-04 B2** - Viktor building Automations 2.0 (Sellers + Buyers) right now. KTS contamination shut down 2026-05-18. Templates 1166 + 1167 created. See `projects/idxre-2026-04.md`
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - blocked on GitHub auth, see `projects/listing-report-portal.md`
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Recently completed

- KTS legacy automation contamination shut down: 6,625 expired leads bulk-applied to `bulk` Automation 2.0, 3 active KTS automations paused at automation level (2026-05-18)
- B2 templates created via FUB API: 1166 (Sellers - Just Checking In) and 1167 (Buyers - Market Shifted), targeting ~693 + ~304 leads (2026-05-18)
- Sellers Guide Meta ads campaign shipped (2026-05-15)
- MLS Grid token wired - CMA Engine auto-pull comps live in production
- Brain App MVP shipped at brain.homegrownpropertygroup.com (2026-05-06)
- Brain App `/api/external/commit` endpoint added for app-code commits to any HGPG1 repo (2026-05-14)
- Brain App `/api/files` reads now require bearer auth (2026-05-15)
- HGPG Listing Reports + MLS Resend SMTP wired (2026-05-15)
- IDXRE-2026-04 B1 campaign scored: 1,289 leads tiered Hot/Warm/Cool/Cold/Dead; 1,284 segment-tagged Seller/Buyer/Unknown; 135 Dead moved to pond 17 IDXRE-Triage (2026-05-17)
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite
- Sellers guide rebrand to brand colors

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- B2 IDXRE launch pending Viktor's automation builds + verification of KTS pause completion (target activation post-verification)
- IDXRE Hot tier outreach PARKED until contamination-aware re-scoring possible
- Stack consolidation parked ~2-3 weeks: kill dead concierge repo, migrate `deals` → `transactions`, fix Title Case vs snake_case status mismatch
- GitHub auth not configured on Mac Mini (blocks Listing Report Portal pushes)
- Sherlock 403 on Transaction Manager (likely API key scope)
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
