# HGPG Context (Brain)

Last updated: 2026-05-07

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
| CMA Engine | 🟡 PRs #16-22 shipped, awaiting Taylor stress test | `projects/cma-engine.md` |
| Transaction Manager | 🟡 ongoing Don feedback batches, $395 fee toggle parked | `projects/transaction-manager.md` |
| TC Concierge | 🟢 live, Don running real deals | `projects/tc-concierge.md` |
| FUB AI Agent | 🟡 Session 2-3 in flight (approval queue, draft gen) | `projects/fub-ai-agent.md` |
| Brain App | 🟢 shipped 2026-05-06, Phase 2 backlog | `projects/brain-app.md` |
| HGPG Team Tools | 🟢 fixed 2026-05-05 (Google OAuth via Supabase) | `projects/team-tools.md` |
| Listing Report Portal | 🟢 GitHub auth resolved | `projects/listing-report-portal.md` |
| Incentives Funnel | 🟢 live, Phase 2 deferred until 50+ leads | `projects/incentives-funnel.md` |
| Claude Skills | 🟢 5 shipped May 1 | `projects/claude-skills.md` |
| Signature + Marketing | (status TBD - verify in file) | `projects/signature-marketing.md` |
| Deals Tracker | (status TBD - likely legacy/decommissioned) | `projects/deals-tracker.md` |
| Main Site SEO | 🟢 mobile PageSpeed 96/100 | `projects/main-site-seo.md` |
| Sellers Guide Meta Pixel + CAPI | 🟡 shipped, QA + Custom Conversion registration pending | (no spec file - tracked in SESSION-HANDOFF) |
| Charlotte New Construction | 🟡 active build, /how-we-work + /incentives shipped | (no spec file - tracked in SESSION-HANDOFF) |
| South Charlotte Report | 🟢 active content pipeline | (no spec file - tracked in SESSION-HANDOFF) |
| Sellers Guide | 🟢 live, rebrand complete | (no spec file - tracked in SESSION-HANDOFF) |
| Buyers Guide | 🟢 live, migrated Manus to React + Vite | (no spec file - tracked in SESSION-HANDOFF) |
| FUB Lead Automation Rebuild | 🔵 planned, foundation ready | (no spec yet) |
| Twilio A2P | ⚫ blocked on SMS consent checkbox | n/a |
| DocuSign migration off zipForms | 🔵 workflow scoped, no build | (chat history) |
| NeverBounce email validation | 🔵 spec written, not built | (chat history) |
| Closing Concierge | 🟢 decommission pending, low priority | n/a |
| $395 fee toggle (TM) | 🔵 build spec parked | (chat history - refs commit b9fa0deb) |

## Recently completed

- Brain App MVP shipped 2026-05-06 (Next.js 16, CodeMirror 6, Supabase magic link, fine-grained PAT for hgpg-context writes)
- HGPG Team Tools auth restored 2026-05-05 (rebuilt on HGPG Core Supabase + Google OAuth)
- Resend custom SMTP wired into HGPG Core Supabase (raises rate limit from 2/hr to 30/hr — affects TM, CMA, TC Concierge, brain-app)
- Supabase project rename to descriptive names (HGPG Core, HGPG Listing Reports + MLS, HGPG FUB Integration, HGPG Signature + Relocation)
- CMA Engine PRs #16-22: outlier auto-exclusion + escape hatch + suppression banner, condition-tier inference, address parser, GLA methodology fix, OUTLIER_MIN_CLUSTER_SIZE lowered 4→3
- Sellers Guide Meta Pixel + CAPI: server-side mirror with event_id dedup, 7 FUB UTM custom fields, UTM-bypass for ?utm_source=meta
- Don's TM feedback Cluster A-D batch shipped (7 items resolved)
- Other-side client email made optional in seller/buyer wizards with format validation
- Favicons rolled out to 9 sites with canonical asset set in brain at assets/favicon/
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routi