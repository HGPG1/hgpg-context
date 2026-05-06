# HGPG Context (Brain)

Last updated: 2026-05-05

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## What's active right now

- **CMA Engine** (cma.homegrownpropertygroup.com) - active build, see `projects/cma-engine.md`
- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, see `projects/transaction-manager.md`
- **TC Concierge** (tc.homegrownpropertygroup.com) - intake, live with Don running real deals
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - GitHub auth now resolved (gh CLI)
- **Charlotte New Construction** (`charlotte-new-construction-nextjs`) - active build
- **South Charlotte Report** content pipeline (`south-charlotte-report` + `south-charlotte-scraper`) - active
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Recently completed

- GitHub auth migrated to gh CLI on Mac Mini (no more embedded PATs)
- Tech & Builds project instructions audited and rewritten against live Vercel + Supabase + GitHub state
- 20+ cruft repos archived (versioned predecessors with `1`, `2`, `3`, `part2` suffixes)
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite
- Sellers guide rebrand to brand colors

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- Sherlock 403 on Transaction Manager (likely API key scope)
- .net Google Workspace migration to .com (do not proactively remind)
- `charlotte-sellers-guide-vercel` vs `charlotte-sellers-guide-2026` — pick canonical, archive the other
- Cruft Vercel projects to clean up: `adoring-wilbur`, `project-lisqf`, `listing-report-deploy`

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
