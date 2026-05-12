<!-- Last Updated: 2026-05-12 -->

# CMA Engine

- **Status:** 🟢 Production-ready, awaiting real-world reps
- **URL:** cma.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-cma-tool (NOT hgpg-cma-engine)
- **Vercel project ID + Supabase IDs:** in memory
- **Supabase architecture (dual-project):**
  - `ioypqogunwsoucgsnmla` (HGPG Core) — `cma_reports` rows (52 total, 47 with narratives, 5 null), RLS open via `cma_reports_open_all` policy
  - `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS) — `mls_property` (2.5M+ rows) for comp search, plus `mls_subject_detect_v2*` RPCs
- **PIN:** 2026

## Current state (2026-05-12)

Engine math sweep complete after three days of intensive bug work (2026-05-07 through 2026-05-12). Math is appraisal-grade per Fannie Mae URAR Form 1004 standards. UX is mobile-responsive. Narrative caching means re-views are instant. Three real-world test properties (Candlestick, Cressingham, Tyndale) verified across three price tiers and three market temperatures.

**The tool is ready for Brenda, Ashley, Taylor to use on real listings.** Pending stress test by Taylor and real-world workflow feedback from agents.

## PRs shipped in this cycle (chronological)

| PR | Description |
|---|---|
| #25 | Bug 5 — PMV bounds invariant (Low ≤ anchor ≤ High; fallback anchor*0.93/1.08) |
| #26 | Bug 6 (active weighting) — Active comp weight drops to 0.03 when only 1 Sold; very-low confidence + banner when 0 Sold |
| #27 | Bug 4 — Anchor sanity check; flag if no comp adjustedPrice within ±5%/$50K |
| #28 | Bug 7 — Subject self-listing exclusion via parsed-parts compare + zip gate |
| #29 | Bugs 8+9 — Distance-tiered cascade (subdivision → 1mi → 3mi → zip) + per-comp distance/direction |
| #30 | Bug 10 — Cross-state deprioritization (0.75× weight + 5pt similarity penalty + amber pill) |
| #31 | Bugs 1+3+11 — Per-feature parity scoring + outlier counter-cluster + strategy band cap |
| #32 | hotfix — Removed below_grade_unfinished_area from COMP_SELECT (column doesn't exist) |
| #33 | Bug 2 — GLA/basement separation per Fannie Mae URAR Form 1004 |
| #34 | Enhancement 1 — Autosave on /seller/adjust + status flip on PDF export |
| #35 | Bugs 6+12+13 — Outlier symmetry + singleton-outlier pass + time-of-sale appreciation |
| #36 | Bug 14 — Appreciation default 5% → 3% + LLM ATTRIBUTION rules (narrative hallucination fix) |
| #37 | Hotfix — Model id swap from deprecated claude-sonnet-4-0 to claude-sonnet-4-6 |
| #38 | Hotfix — Surface upstream Anthropic error + bump TIMEOUT_MS 60s → 120s |
| #39 | Enhancement 2 (incomplete first pass) — Mobile responsive (wrap-to-second-row header) |
| #40 | Enhancement 2 followup — Hamburger drawer + input width fix at source + removed overflow-x-hidden band-aid |
| #41 | Polish — Address normalization (canonical from MLS structured parts; mls_subject_detect_v2 RPCs return parts) |
| #42 | Polish — Narrative caching (narrative_generated_at column + trigger + loadCachedNarrative) |
| #43 | Polish — Smart freshness trigger so non-math UPDATEs preserve narrative cache |

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
- Strategy band cap (PR #31, Bug 11): Aspirational ≤ High, Event ≥ Low. Strategies always inside the confidence band.

### Hard rule

**Never use "fair market value."** Always use "Perceived Market Value" / PMV.

### GLA + basement adjustment rates (PR #33, Bug 2)

Per Fannie Mae URAR Form 1004 standards:
- **GLA (above-grade finished area):** $60/sqft contributory value (paired-sales-derived for Charlotte-metro $500K-$900K)
- **Finished basement:** $25/sqft
- **Unfinished basement:** $10/sqft

Carolina MLS exposes `above_grade_finished_area` and `below_grade_finished_area`. `living_area = above_grade + below_grade` (SUM, includes basement). **`below_grade_unfinished_area` does NOT exist** — comp-side unfinished basement is unrecoverable from feed; subject-side is manual input only.

NOT full price-per-sqft — that double-counts pool, garage, lot, condition, etc. and fails Fannie Mae CU review.

### Time-of-sale appreciation (PR #35, Bug 13 + PR #36, Bug 14)

Calibrated to Charlotte metro 2024-2025 cooling market:
- **Default appreciationRatePerYear: 3.0%**
- 0-90 days old: no adjustment
- 91-180 days old: +1.5% (half rate) applied to raw price before other adjustments
- 181-365 days old: +3.0% (full rate)
- >365 days old: `stale-sale` flag, exclude from math entirely

Tunable per-deal in /seller/adjust Rates panel. Hot submarkets (Fort Mill, Waxhaw, Ballantyne) may use 4-5%. Cooling submarkets (Lancaster County, eastern Union) often 0-2%. Real data driving the 3.0% default:
- Indian Land Q4 2024: +2.9% YoY (Rocket Homes)
- Lancaster County Jul 2025: -1.1% YoY (Redfin)
- Indian Land Apr 2026: -1.0% YoY (Movoto)

### Outlier protection (PR #31, #35)

Three-layer outlier handling:
1. **Singleton outlier** (Bug 12): cap weight at 0.25× when adjustedPrice >50% above OR >40% below other-closed median AND similarity <0.65 AND ≥1 quality flag (long-market DOM>90, PQS<50, sqft-mismatch, pool-mismatch, cross-state). Counter-cluster-protected comps skip the cap.
2. **Counter-cluster protection** (Bug 11): pairs of flagged comps on same side of anchor get pair-penalized at 0.5× rather than auto-excluded if cluster median is ≤25% from anchor.
3. **Outlier symmetry** (Bug 6, refactored in PR #35): scans each flag for unflagged near-twin (5% price proximity, same side, similarity within 0.10). Cluster median >25% from anchor → both flagged at 0.5×; ≤25% → both clean.

### Cross-state policy (PR #30)

Cross-state comps stay visible but rank lower:
- 0.75× weight penalty
- 5pt similarity penalty
- Amber "cross-state" pill on the comp card
- Underwriters push back because effective property tax (NC ~0.7-0.85% vs SC ~0.5-0.6%), income tax, transfer-tax, school district differ at the border

### Subject self-listing exclusion (PR #28)

Parsed address parts compare + zip gate, suffix-tolerant. Prevents subject from appearing as its own comp.

### Narrative caching (PR #42, #43)

- `cma_reports.narrative_generated_at` timestamptz column
- `cma_reports_set_updated_at` trigger is smart about freshness:
  - Math changed (`subject_summary`/`comps`/`adjustments`/`pmv`/`dom`/`pqs`) → leave `narrative_generated_at` alone → stale signal (intentional)
  - Narrative changed → stamp `narrative_generated_at = updated_at`
  - Neither changed, row already has narrative (status flip, label edit) → bump along with `updated_at` → cache stays valid
- `lib/cma/narrative-cache.ts → loadCachedNarrative(reportId)` returns saved narrative when fresh, null otherwise
- Three packet pages (seller/buyer/appraiser) check cache on mount, skip Claude call if hit
- Regenerate button always forces fresh Claude call (escape hatch)
- Future writers to `cma_reports` don't need to remember to bump `narrative_generated_at` — trigger handles it

### Autosave (PR #34)

- 500ms debounced writes on every edit to /seller/adjust (rate, weight, comp swap, condition tier, agent inclusion)
- editVersion counter bumps on every state change
- 3x exponential backoff retry on transient failures
- "Draft — Saving / Saved HH:MM / Save failed" banner under page eyebrow
- First save INSERTs and captures returned id back into WorkingSet; subsequent autosaves UPDATE same row
- Continue-to-Packet flushes in-flight save before navigating
- Status flips from 'draft' to 'presented' on successful PDF export (not button click — that's the natural "deliverable is done" moment)
- Status enum: `draft | presented | archived`

### Address normalization (PR #41)

- `mls_property.unparsed_address` is NULL across the entire 2.5M-row dataset — confirmed via verification
- Canonical display address constructed from `street_number + street_name + street_suffix + unit_number` — all reliably populated, all in proper case with full-word suffixes
- `lib/cma/addressNormalize.ts` exposes:
  - `buildCanonicalAddress(parts)` for MLS-match path
  - `normalizeAddress(raw)` for manual entry / legacy reports (idempotent, handles "215 tyndale ct" → "215 Tyndale Court" and edge cases like "5601 N Main St" → "5601 North Main Street", "215 Dr Phil Lane" left alone)
- Three `mls_subject_detect_v2*` RPCs migrated to additively return the four parts fields at end of result tuple
- Defensive normalize at `loadWorkingSet` cleans the 43+ legacy `cma_reports` rows on history reopen — no in-place backfill needed

### Mobile layout (PR #40)

- Header is a client component with responsive split: at md+ inline nav unchanged; below md, logo left + hamburger right with right-side drawer (48px tap targets, backdrop/link/ESC close, body scroll-lock, inline SVG icons)
- Form inputs use `block w-full` with `min-w-0` on label wrappers — fixes overflow at source rather than masking with body `overflow-x-hidden`
- White card padding shrunk from `p-10` to `p-6 sm:p-10` (24px mobile, 40px desktop)

## Real-world verification (2026-05-11/12)

Three test properties verified across price tiers and market temperatures:

**Candlestick (6022 Candlestick Ln, Bent Creek SC, luxury 6BR)**: PMV $1,020,505, LOW (manual review) confidence. Wyngate-anchored cluster math from 5 remaining comps after Cutter Court correctly excluded as +111% singleton outlier. Aspirational $1,072K / PMV $1,021K / Event $969K all inside band.

**Cressingham (5022 Cressingham, Indian Land SC, 5BR ranch)**: PMV $670,983, MODERATE confidence. Five same-street Solds. Aspirational $700K / PMV $671K / Event $652K. Tight cluster, no exclusions, narratives correctly attribute time-of-sale (not "newer vintage" hallucination).

**Tyndale (215 Tyndale Ct, Somerset 28277)**: PMV $788K, tight confidence inferred from 4% spread. Aspirational $818K / PMV $788K / Event $767K. Hot market — narrative correctly engages with "median DOM is 2 days" context.

## Key engineering rules captured

1. **Dual-Supabase architecture**: cma_reports lives in HGPG Core; MLS data lives in Listing Reports. Code memory: "diagnose, don't create."
2. **Verify Postgres columns before specifying** (lesson from PR #32 hotfix).
3. **Carolina MLS field names** verified: above_grade_finished_area, below_grade_finished_area, living_area = SUM. No below_grade_unfinished_area column.
4. **Status enum is `draft | presented | archived`** — not 'published'.
5. **Appreciation defaults must reflect current market era** (3.0% for Charlotte metro 2025; tunable per-deal).
6. **LLM narrative hallucination** prevented via per-comp line breakdown + explicit ATTRIBUTION rules in system prompt.
7. **Model IDs go stale** — prefer latest-version alias (claude-sonnet-4-6) over snapshot dates.
8. **Database triggers > per-call-site discipline** for cross-cutting concerns like cache invalidation.

## What's deferred (not blockers)

- **Build-year premium**: dropped intentionally. Charlotte metro doesn't reliably price 2019 vs 2020 differently.
- **Time-on-market adjustment for actives**: different from Bug 13's closed-comp appreciation. Lower priority.
- **Submarket-specific appreciation rates**: real-world workflow item. Agents tune per-deal in Rates UI rather than coding submarket logic.
- **Soft Shell pair symmetry verification** on May 7 Candlestick run: low priority since the May 7 draft is mostly archival now.

## Pickup checklist

1. Use the tool on a real listing this week. Let real-world friction surface real priorities.
2. Taylor stress test (was the original pending item before this cycle started).
3. Brenda/Ashley/Don feedback as they use it on their own deals.
4. Other repos for CLAUDE.md ship-it specs (deferred until CMA gets real-world reps): hgpg-transaction-manager (highest impact next), brain-app, charlotte-new-construction-nextjs, south-charlotte-report, homegrown-property-group-site.

## Live test reports (regression baselines)

- `7cb7ef75-6c96-4454-baf6-c94265e91a4f` — current Candlestick draft, latest math
- `a4283aa0-68ad-42c7-8258-90a372698d5b` — current Tyndale draft
- Historical: `b4278470`, `3a84f12c`, `8023175d` (Candlestick variants); `5db8a5c7`, `e6eae387`, `fccfa5ba` (Tyndale variants); Cressingham 5022; 504 Redwine; 5601 Medlin
