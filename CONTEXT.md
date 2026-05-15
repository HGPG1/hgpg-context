# HGPG Context (Brain)

Last updated: 2026-05-15

## Naming convention

- **"NC"** = North Carolina (the state). Always.
- **"New Construction site"** = `newconstruction.homegrownpropertygroup.com`. Never abbreviate as "NC site," "NC Scout," "NC ad," etc. Spell it out every time. The state and the product line share too much surface area.

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists), indoor air quality monitoring

## What's active right now

- **CMA Engine** (cma.homegrownpropertygroup.com) - 🟢 production-ready, math sweep complete 2026-05-12, awaiting real-world reps from agents. QA checks are #1 in current priority queue. See `projects/cma-engine.md`
- **Transaction Manager** (closings.homegrownpropertygroup.com) - 🟡 active build, ongoing Don feedback. See `projects/transaction-manager.md`
- **TC Concierge** (tc.homegrownpropertygroup.com) - 🟢 live, Don running real deals. See `projects/tc-concierge.md`
- **Team Photo Sync v1** (cron in hgpg-cma-tool) - 🟢 LIVE 2026-05-15. Rehosts team MLS photos to public Supabase Storage every 30 min. 1,591/1,653 backfilled, see `projects/team-photo-sync.md`.
- **Team Dashboard** (team.homegrownpropertygroup.com/inventory) - 🟢 hero photos now wired to mls_media.media_url_rehosted
- **New Construction site** (newconstruction.homegrownpropertygroup.com) - 🟢 three production deploys shipped 2026-05-13 (phone capture tiering, SMS speed-to-lead, Scout admin cleanup). SMS speed-to-lead pending live end-to-end test from Brian's real phone. See `projects/new-construction-*.md` cluster.
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com) - 🟢 shipped + ad-ready. FUB Automation 2.0 parked for first real ad lead validation. Meta ad launch is #2 in priority queue. See `projects/sellers-guide.md`
- **Buyers Guide** (buyersguide.homegrownpropertygroup.com) - 🟢 Manus migration Sessions 1+2 complete 2026-05-12, Sessions 3-4 queued. See `projects/buyers-guide.md`
- **Brain App** (brain.homegrownpropertygroup.com) - 🟢 MVP shipped 2026-05-06, single-user lock. Backed by GitHub App **HGPG Brain Commit** (installed 2026-05-14, read+write across all HGPG1 repos, no PAT rotation needed). `/api/external/commit` endpoint allows external commits but cannot delete files - real deletes need `gh` from the Mac. See `projects/brain-app.md`
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - 🟢 live, GitHub auth resolved 2026-05-06. See `projects/listing-report-portal.md`
- **HGPG FUB MCP** (fub.homegrownpropertygroup.com/mcp) - 🟢 hosted remote MCP live with OAuth 2.0 + PKCE + DCR, 134 FUB tools in safe mode. Tier 2 workflow-layer tools is #4 in priority queue.
- **Signature Marketing Collection** (signature.homegrownpropertygroup.com) - 🟢 live. See `projects/signature-marketing.md`
- **Property Marketing Analyzer** (marketinganalyzer.homegrownpropertygroup.com) - 🟢 live, separate tool
- **Claude skills** - five shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Current priority queue (2026-05-14)

1. CMA Engine QA checks (scope TBD)
2. Launch Meta ad for Sellers Guide
3. Review/check ads for New Construction site
4. Build Tier 2 FUB MCP - workflow-layer business tools + `fub_agent_*` Supabase integration

## Parked - run on a date

- **New Construction site phone capture rate review** - run between 2026-05-19 and 2026-05-26, see `projects/new-construction-phone-capture-rate-review.md`. Pull FUB data after 7-14 days post-shipping, evaluate if optional-phone nudges are pulling weight, decide if iteration needed.
- **New Construction site "For Builders" footer link** - low priority, bundle with next New Construction site touch, see `projects/new-construction-builder-submit-footer-link.md`. Adds a discreet footer link to /builder-submit so builder reps can find the incentive submission form without Brian manually sharing the URL.

## Recently completed

- **Team Photo Sync v1 ship** (2026-05-15) - production cron rehosting MLS team photos to public Supabase Storage. See `projects/team-photo-sync.md`.
- **New Construction site Scout admin cleanup** (2026-05-13) - scout_status column on nc_builders, Scout admin tab filters to crawl-only (23 -> 19), KB Home URL fix, 4 builders flipped to email_only. See `projects/new-construction-scout-admin-cleanup.md`.
- **New Construction site SMS speed-to-lead** (2026-05-13) - Builder Intro instant SMS via Lead Flow + 5-min backup task via Automation 2.0, see `projects/new-construction-sms-speed-to-lead.md`. Live end-to-end test pending.
- **New Construction site phone capture** (2026-05-12) - tiered downgrade, see `projects/new-construction-phone-capture.md`
- **CMA Engine math sweep** (2026-05-07 to 2026-05-12) - 7 PRs (#25-#31) fixing Bugs 1, 3-11 to Fannie Mae URAR Form 1004 standards
- **HGPG FUB MCP migration to hosted remote** (2026-05-14) - OAuth 2.0 + PKCE + DCR, 134 FUB tools in safe mode, accessible from web/Desktop/mobile without local Node
- **GitHub App "HGPG Brain Commit" installation** (2026-05-14) - read+write across all HGPG1 repos, replaces PAT-based auth for the Brain App
- **Brain App MVP** (2026-05-06)
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite (Sessions 1+2)
- Sellers guide rebrand to brand colors

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- **Live end-to-end test for New Construction site SMS speed-to-lead** - Brian needs to submit a Builder Intro from incognito with his real phone to verify
- Sherlock 403 on Transaction Manager (likely API key scope) - parked, do not resurface unless Brian raises it
- .net Google Workspace migration to .com (do not proactively remind)

## Resolved (do not resurface)

- GitHub auth on Mac Mini (resolved 2026-05-06)
- GitHub PAT exposure - all old PATs revoked during context repo migration; sandbox cannot push to GitHub
- CMA Engine Lamington duplicate row (fixed)
- FUB stdio MCP (superseded 2026-05-14 by hosted remote MCP)

## Standing rules learned this week

- **MLS Grid v2 has many undocumented limits.** See `projects/team-photo-sync.md` for the full list. Highlights: `$expand=Media` needs `$select` + `$top=50` to stay under response cap. Property `$filter` only accepts 7 specific fields. Nested Media has wrong `ResourceRecordKey` and missing `OriginatingSystemName` — always override from parent context.
- **MLS Grid media CDN (`media.mlsgrid.com`) is rate-limited separately** (~7-10 RPS). Always throttle between rehost downloads (150ms is enough).
- **PostgREST `.in()` has URL length limits.** Chunk to 200 keys/batch.
- **Supabase `upsert()` clobbers ALL columns by default.** Preserve via lookup-and-merge.
- **Vercel build SIGTERM** = a route is doing outbound HTTP at build time during prerender. Add `export const dynamic = 'force-dynamic';` to any route that fetches external data.
- **Burst-debugging crons is a foot-gun.** Hitting an endpoint 13x in 60s to "drain faster" recreates rate-limit problems. Trust the production schedule.
- **Brain App's `/api/external/commit` cannot delete files** — only stub them. Real deletes need `gh` from the Mac.
- **FUB Lead Flow conditions are restricted** to: Tags, Price, City, State, ZIP, MLS, Phone Number. Custom fields are NOT filterable at Lead Flow.
- **FUB Automations 2.0 have NO Send Text step.** Only Lead Flow can send native auto-SMS.
- **FUB custom field API names preserve uppercase runs in labels.** Always GET /customFields to verify after creating.
- **FUB send-from number for new construction:** (980) 261-9222
- **555-pattern phones get flagged invalid in FUB** and won't actually receive SMS. Use real phones for tests.
- **Builder-rep submission page lives at `/builder-submit`** on the New Construction site (unlinked from nav by design - shared back-channel for now)
- **New Construction site Scout: builders fall into 3 ingestion buckets** - crawl (plain HTTP works), email_only (JS-rendered or WAF-blocked, ingested via Apps Script email pipeline), paused (not pulling). Filter Scout admin tab by scout_status. Filter at API layer, not just UI.

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
