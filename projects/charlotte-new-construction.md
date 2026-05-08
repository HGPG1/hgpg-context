<!-- Last Updated: 2026-05-08 -->

# Charlotte New Construction

- **Status:** 🟡 Active build
- **URL:** newconstruction.homegrownpropertygroup.com
- **Repo:** HGPG1/charlotte-new-construction-nextjs
- **Default branch:** `main`
- **Stack:** Next.js (server-rendered), Vercel deployment
- **Vercel team:** `team_FietQPKCmnyioG2n0FdteQCV`

## Purpose

Lead-gen funnel for buyers interested in new-construction homes in the Charlotte metro area. Drives Meta paid traffic to landing pages, captures leads with progressive disclosure, syncs to FUB.

## Routes shipped

| Route | Purpose | Lead capture |
|---|---|---|
| `/` | Main landing | Yes |
| `/incentives` | Builder incentive comparison + lead form | Yes (primary funnel) |
| `/how-we-work` | Service explainer | Yes |
| `/quiz` | Buyer match quiz | Yes |

All 4 routes have FUB lead capture wired.

## Meta Pixel + CAPI

- Pixel ID: `1880396459290092`
- CAPI live with browser+server dedup
- All 6 funnel events firing (PageView, ViewContent, Lead, etc.)
- Pattern documented in `META-PIXEL-CAPI-PLAYBOOK.md` in repo root — reusable for other sites; each site needs its own Pixel ID

## FUB integration

Lead form on every route POSTs to a server-side route handler that pushes to FUB with:
- Source: `Charlotte New Construction`
- UTM custom fields populated when present in URL
- Lead routing per FUB Automations 2.0 rules (NOT Action Plans, those are deprecated)

## Open / pending

- **NeverBounce email validation** for the `/incentives` form — spec written in `projects/neverbounce-validation.md`, not yet built
- **Phase 2 ad creative** still in production with Sami
- **Builder data** for the incentive comparison page is hardcoded — not pulled from a CMS yet. When the third or fourth builder gets added we'll want a CMS or a JSON config file with rotation logic
- **Quiz scoring logic** is functional but the buckets haven't been calibrated against actual lead quality yet (need data from first 50 leads)

## Key learnings

- Meta Pixel + CAPI playbook in repo root is the reusable template for any other HGPG site that needs the same setup. Each site needs its own Pixel ID (you cannot share one Pixel across multiple domains effectively)
- Server-side rendering (Next.js App Router) means client-side `fbq()` events fire in tight succession on initial render — important to make sure dedup `event_id` is generated correctly so the CAPI mirror doesn't double-count

## Pickup notes

- This is one of the actively-marketed funnels — Meta ad budget runs through here weekly
- Repo has a `META-PIXEL-CAPI-PLAYBOOK.md` at root that should be referenced (not duplicated) by future sites adopting the pattern
- When NeverBounce validation ships, this is the first form it gets wired to (per spec in `projects/neverbounce-validation.md`)
