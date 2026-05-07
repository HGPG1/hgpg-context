<!-- Last Updated: 2026-05-07 -->

# CMA Engine

- **Status:** 🟡 Active build, awaiting Taylor stress test
- **URL:** cma.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-cma-tool (NOT hgpg-cma-engine)
- **Vercel project ID + Supabase ID:** in memory
- **Supabase:** `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS) - tables for CMA + MLS replication
- **PIN:** 2026

## Architecture decisions

### Status-weighted Perceived Market Value (PMV)

- Solds: **60%**
- Under Contract - No Showing pendings: **25%**
- Under Contract - Showing pendings: **10%**
- Actives: **5%**

### Sharran Srivatsaa three-strategy pricing framework

| Strategy | Price | Notes |
|---|---|---|
| Aspirational | PMV + 3-7% | |
| PMV anchor | PMV | |
| Event | PMV - 2-5% | |

- Verbatim scripts required.
- **Sharran attribution required** in output.

### Hard rule

**Never use "fair market value."** Always use "Perceived Market Value" / PMV.

### GLA adjustment rate

- 60 $/sqft contributory value (paired-sales-derived for Charlotte-metro $500K-$900K)
- NOT full price-per-sqft - that double-counts pool, garage, lot, condition, etc. and fails Fannie Mae CU review

## Data sources

- **Primary:** MLS Grid -> Supabase replication (Canopy MLS Brokerage Back Office license, `OriginatingSystemName='carolina'` lowercase)
- **Replication cron:** `*/15` at `/api/cron/mls-sync`
- **Search function:** `searchComps()` in `lib/mls/compSearch.ts`
- **Fallback:** IDX Broker `/mls/search` retained behind `MLS_USE_IDX_FALLBACK=true` flag (kill-switch only)
- **Manual comp entry:** still supported via Adjust UI

## Outlier handling (PR #14-17)

- `OUTLIER_PCT_THRESHOLD` = 0.20 (20% above cluster median triggers outlier flag)
- `OUTLIER_MIN_CLUSTER_SIZE` = 3 (lowered from 4 in PR #17 to match appraisal-standard minimum)
- Auto-exclusion when cluster size after exclusion >= floor
- Escape hatch: agent can re-include via "Include in math" link
- Suppression banner shown when floor blocks auto-exclusion

## Key files

- `lib/cma/outlierFlag.ts` - threshold + cluster floor constants
- `lib/cma/pmv.ts` - `computePmv` returns `{ autoExcludedCompIds, exclusionSuppressedReason, outlierFlags, ... }`
- `lib/cma/store.ts` - `WorkingSet` shape (includes `agentOverrides.includedExcludedCompIds`, `pool`, `manualSwaps`)
- `app/seller/adjust/page.tsx` - banner, exclusion badge, "Include in math" link, comp adjustment table
- `app/api/packet/seller/route.ts` and `buyer/route.ts` - LLM prompt context (passes exclusions to LLM)
- `scripts/test-ranking.ts` - 146 unit fixtures, all pass

## Recent ships

### April 19, 2026 build session (7 commits)

1. GLA methodology fix
2. Neighborhood bracketing
3. Duplicate row bug fix on reopen-and-regen
4. History filter
5. DOM fix
6. Methodology doc comments
7. Renamed "Value-Defense Report" -> "Supporting Market Analysis"

### Late April / Early May 2026 (PRs #14-22)

- **PR #14:** Outlier flagging system (introduced `OUTLIER_MIN_CLUSTER_SIZE=4`)
- **PR #15:** WorkingSet pool + manualSwaps support
- **PR #16:** Auto-exclusion + escape hatch + suppression banner
- **PR #17:** Floor lowered 4 -> 3 (appraisal-standard minimum, real-world test case 3439 Twelve Oaks Place exposed floor=4 as too conservative)
- **PR #18-21:** Condition-tier inference, address parser, narrative bug fixes
- **PR #22:** Favicon rollout

### MLS Grid sync (PR #4)

- 9 tables, replicated from Canopy MLS
- `mls-grid-sync` branch -> main
- Default comp filters exclude lease/rental and non-residential subtypes
- IDX Broker path retained as kill-switch behind `MLS_USE_IDX_FALLBACK=true`

## Workflow conventions

- Send Claude Code prompts in single triple-fence blocks; never nest triple-backticks - use 4-space-indented code samples inside the outer fence
- Each Claude Code session: `cd ~/Documents/hgpg-cma-tool && git checkout main && git pull`, then `claude --dangerously-skip-permissions`, then paste prompt
- Vercel build cache occasionally serves stale compilations; if a fix doesn't take effect on first deploy, push an empty commit (`git commit --allow-empty -m "chore: force rebuild"`) BEFORE debugging logic
- Commit author: `brian@homegrownpropertygroup.com` (NOT `brian@hgpg.com`)
- Floating feedback widget on the CMA tool (PR #10) captures full WorkingSet JSON to Brian's iMessage if anything breaks

## Open items

- Awaiting Taylor stress test on real listings before declaring engine production-ready
- MLS Grid auto-pull comps: pending Canopy MLS Bearer token from Bridgett Bouvier
- Saved reports snapshot `ratesUsed` at save time; "Regenerate" re-runs AI narrative but does NOT recalculate math. To recalculate, must open Adjust screen for that specific report, confirm/change rate, save explicitly.

## Known constraints

- IDX Broker `/mls/search` endpoint is NOT enabled on current tier - hence the MLS Grid replication path
- HGPG = "the team" or "Home Grown Property Group" - never "the brokerage" (Real Broker LLC = brokerage)
