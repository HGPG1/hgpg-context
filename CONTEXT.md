# HGPG Context (Brain)

Last updated: 2026-05-11 (NC Incentives lead quality audit, ads paused)

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## Active projects

Status legend: 🟢 shipped & live · 🟡 active build · 🔵 planned/scoped · ⚫ blocked · 🔴 broken

| Project | Status | Spec |
|---|---|---|
| CMA Engine | 🟡 PRs #16-22 shipped, MLS Grid integration mature, awaiting final stress test | `projects/cma-engine.md` |
| Transaction Manager | 🟡 ongoing Don feedback batches, transaction-pdfs bucket flip in flight | `projects/transaction-manager.md` |
| TC Concierge | 🟢 live, supports TM data path (not directly used anymore) | `projects/tc-concierge.md` |
| FUB AI Agent | 🟡 Viktor's Automation 2.0 complete, smoke test gate pending | `projects/fub-ai-agent.md` |
| Brain App | 🟢 shipped 2026-05-06 + write API live 2026-05-08 (`/api/external/write`) | `projects/brain-app.md` |
| Team Dashboard | 🟢 Tab 1 + Tab 2 shipped 2026-05-08 | `projects/team-dashboard.md` |
| HGPG Team Tools | 🟢 fixed 2026-05-05 (Google OAuth via Supabase) | `projects/team-tools.md` |
| Listing Report Portal | 🟢 stable, in production | `projects/listing-report-portal.md` |
| Incentives Funnel | 🟢 live, Phase 2 deferred until 50+ leads | `projects/incentives-funnel.md` |
| Claude Skills | 🟢 5 shipped May 1 | `projects/claude-skills.md` |
| Signature + Marketing | 🟢 live, dark luxury editorial site + Property Marketing Analyzer | `projects/signature-marketing.md` |
| Main Site SEO | 🟢 mobile PageSpeed 96/100 | `projects/main-site-seo.md` |
| Sellers Guide | 🟢 live, rebrand complete + Home Grown Selling Score v2 shipped 2026-05-08 | `projects/sellers-guide.md` |
| Home Grown Selling Score | 🟢 live 2026-05-08 (5x4 wizard, curved 0-80 score, lead capture at end) | `projects/sellers-guide.md` |
| Sellers Guide Meta Pixel + CAPI | 🟢 shipped, Custom Conversion registration still pending | `projects/sellers-guide.md` |
| Charlotte New Construction | 🟡 site live, Meta ads PAUSED 2026-05-11 (lead quality fix in HGPG-Ads), Pixel/CAPI load-bearing | `projects/charlotte-new-construction.md` |
| South Charlotte Report | 🟢 active content pipeline | `projects/south-charlotte-report.md` |
| Buyers Guide | 🟡 rebuild well under way (was 🟢 live on Manus→React+Vite) | `projects/buyers-guide.md` |
| NC Scout / Incentive Concierge | 🟢 live, dashboard hide-failed-syncs follow-up scoped | `projects/nc-scout.md` |
| PropStream Caller | ⚫ parked / mostly abandoned, status unclear | `projects/propstream-caller.md` |
| Twilio A2P | ⚫ blocked on SMS consent checkbox | n/a |
| DocuSign migration off zipForms | 🔵 workflow scoped, no build | `projects/docusign-migration.md` |
| NeverBounce email validation | 🟢 live on /incentives + Home Grown Selling Score, Phase 2 rollout pending | `projects/neverbounce-validation.md` |
| $395 fee toggle (TM) | 🔵 build spec parked | `projects/tm-395-fee-toggle.md` |

## Recently completed (latest first)

- **2026-05-11 NC Incentives Meta campaign lead quality audit.** Pulled 16 FUB leads via MCP, ran geo audit on socialData. ~94% out-of-market (NJ/VA/NY/PA/TX/CA), only 1 confirmed in-market (Tammy Flores, FUB 31833). Real in-market CPL = ~$318, not the $12 Meta reports. Root cause: Housing Special Ad Category strips standard targeting levers + default "People in or recently in" location type. Ads paused pending creative-led fix in HGPG - Ads project. Variant C "Choice" ($50K OR rate) decisively wins on quality metrics, $4.63 CPL, 2.60% CTR, 33% LP-to-lead, Above avg rankings, keep this pattern for Variant D creative. Full audit lives in `projects/charlotte-new-construction.md`.
- **2026-05-09 PM session reconciled status across all projects** (this update). Closed: Sherlock 403, Lamington duplicate cleanup, Google Calendar OAuth, FUB API key swap, Suna teardown, Closing Concierge teardown verified. Scope cuts: Meta Pixel rollout to Marketing Analyzer + TM both NOT NEEDED.
- **TM transaction-pdfs bucket flip → private + signed URLs** kicked off as Claude Code task 2026-05-09. Spec covers conciergePdf rehydrated URLs, send-task-email rehydrate, and rezen push-document delete path.
- **MLS Grid auto-pull comps integration mature** in CMA Engine. Bridgett's Bearer token landed; primary data source for `searchComps()`.
- **NeverBounce wired into Home Grown Selling Score v2 (2026-05-08)**, mirrors the implementation on charlotte-new-construction-nextjs /incentives. New /api/validate-email serverless function, debounced 500ms blur validation with inline UI feedback, hard-block on invalid+disposable, allow-with-warning on catchall+unknown, fail-open on errors. Supabase migration `seller_assessments_add_email_validation_2026_05_08` adds 2 columns + partial index. FUB customEmailValidationStatus custom field reused.
- **Home Grown Selling Score v2 shipped 2026-05-08**, replaced 46-item wizard with 5 categories × 4 items, internal 4/2/1/-1 scoring (max raw 80), client-facing 0-80 curved score (no flat ceiling, no perfect achievable). New `/api/assessment/submit` endpoint, Supabase migration `seller_assessments_v2_2026_05_07`, all Meta Pixel anchors preserved. FUB push via existing `/api/fub-lead`.
- **Brain App write API live 2026-05-08**, `/api/external/write` POST endpoint with bearer token auth allows Claude sessions to commit directly to `HGPG1/hgpg-context` without manual copy-paste. Token stored in Vercel as `BRAIN_WRITE_TOKEN`. Fix in commit `127cc0c` corrected env var lookup to use `GITHUB_PAT` (matches what brain-app already uses).
- Brain App MVP shipped 2026-05-06 (Next.js 16, CodeMirror 6, Supabase magic link, fine-grained PAT for hgpg-context writes)
- Team Dashboard Tab 1 + Tab 2 shipped 2026-05-08
- HGPG Team Tools auth restored 2026-05-05 (rebuilt on HGPG Core Supabase + Google OAuth)
- Resend custom SMTP wired into HGPG Core Supabase (raises rate limit from 2/hr to 30/hr, affects TM, CMA, TC Concierge, brain-app)
- Supabase project rename to descriptive names (HGPG Core, HGPG Listing Reports + MLS, HGPG FUB Integration, HGPG Signature + Relocation)
- CMA Engine PRs #16-22: outlier auto-exclusion + escape hatch + suppression banner, condition-tier inference, address parser, GLA methodology fix, OUTLIER_MIN_CLUSTER_SIZE lowered 4→3
- Sellers Guide Meta Pixel + CAPI: server-side mirror with event_id dedup, 7 FUB UTM custom fields, UTM-bypass for ?utm_source=meta
- Don's TM feedback Cluster A-D batch shipped (7 items resolved)
- Other-side client email made optional in seller/buyer wizards with format validation
- Favicons rolled out to 9 sites with canonical asset set in brain at assets/favicon/
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing live, search.homegrownpropertygroup.com on Realty Candy / Canopy MLS, IDX Addons widget on main site)

## Pinned / strategic

- **Consolidation opportunity: Listing Report Portal + Team Dashboard + CMA Engine.** All three lean on the same MLS Grid replication and PIN/magic-link auth. Real argument for one unified "HGPG MLS Workspace" with tabs. Counterargument: seller-facing polish bar differs from internal tooling. Park for a strategic review session — no work yet.
- **NewCon serving live ads** — periodic Pixel/CAPI health checks in Meta Events Manager warranted. Silent break = wasted ad budget.

## Rainy day items

- **Vercel project cleanup audit.** 8+ candidates appear dormant: hgpg-transaction-monitor, fub-texting-integration, hgpg-listing-admin, hgpg-sites-dashboard, hgpg-blog-dash, hgpg-blog-automation-dashboard, hgpg-calc, hgpg-lead-caller, charlotte-neighborhood-guides. Plus DNS cleanup in GoDaddy — remove `concierge` CNAME (subdomain torn down).

## Decommissioned / done

- Suna (Supabase project `nfwjgsanvmwmrginhklz`) — fully torn down 2026-05-04
- Closing Concierge (`closing-concierge` repo, `concierge.homegrownpropertygroup.com`) — Vercel project removed, DNS CNAME cleanup pending
- Twilio (fully deprecated 2026-04, LoopMessage is sole TM messaging path)
- FUB Action Plans (deprecated, Automations 2.0 only)
- hgpg-deals-tracker (deals.homegrownpropertygroup.com) — decommissioned 2026-05-01, all data mirrored into transactions
