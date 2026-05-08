<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-08 (overnight) — CMA PR #33 GLA / basement separation 🟢

### What shipped

PR #33, GLA / basement separation per Fannie Mae URAR Form 1004. https://github.com/HGPG1/hgpg-cma-tool/pull/33

The engine was comparing `subject.sqft` (total) against `mls_property.living_area` (total = above + below) at the GLA rate, double-counting basement and mis-adjusting on every comp with one. URAR requires GLA = above-grade only with separate contributory rates for finished and unfinished basement. Carolina MLS exposes this structurally via `above_grade_finished_area` + `below_grade_finished_area` (PR #31 already wired the columns into CompRow).

- **Subject form:** three numeric inputs (GLA, finished basement, unfinished basement) plus read-only Total replace the legacy single Heated sqft field. Every breakdown edit recomputes `subject.sqft = sum` so legacy ranking / flags / packet routes keep reading the same field. MLS auto-fill populates GLA + finished basement directly from the structured columns.
- **Adjustment engine:** single Square Footage line replaced with up to three lines (GLA / Finished basement / Unfinished basement). GLA reads `comp.hgpgGlaSqft` (from `above_grade_finished_area`) with fallback to `sqftOf` plus a new `estimated-gla` info flag when the structured column is missing. Finished basement reads `comp.hgpgFinishedBasementSqft`. Unfinished basement is subject-side only because MLS Grid sync doesn't carry a `below_grade_unfinished_area` column - documented limitation.
- **Rates:** new `finishedBasementRate` ($25/sqft default) + `unfinishedBasementRate` ($10/sqft default) on AdjustmentRates. Optional on the type for back-compat with older `ratesUsed` snapshots.
- **Worked example (Wyngate vs Candlestick):** pre-fix Wyngate adjustment was a single +$6,000 line on a 100 sqft delta. Post-fix per-line: GLA +$32,340, Finished basement -$10,975, Unfinished basement +$10,000 = +$31,365. PMV anchor expected to rise ~$15-30K on Candlestick.

### Cumulative CMA ship list this session window

#25 Bug 5 bounds invariant, #26 Bug 6 active weighting, #27 Bug 4 anchor sanity, #28 Bug 7 self-listing suffix, #29 Bugs 8+9 distance-tiered cascade + per-comp distance/direction, #30 cross-state comp deprioritization, #31 math bundle (feature parity / outlier symmetry / strategy band cap), #32 hotfix drop unfinished column from COMP_SELECT, #33 GLA / basement separation. All live on `cma.homegrownpropertygroup.com`.

### Pickup notes (CMA)

- PR 6 (Bug 3, outlier symmetry) is now LOWEST priority - already partly addressed by PR #31's counter-cluster protection.
- Enhancement 1 (autosave) still queued as a separate piece of work.
- Saved test reports still hold pre-fix numbers; re-saving via `/seller/adjust` is required to apply the cumulative changes to historical rows.

---

## Earlier session: 2026-05-08 (PM) — Team Dashboard Tab 1 SHIPPED 🟢

### What got shipped
- New repo live: `HGPG1/hgpg-team-dash` (commit `e9d1366` initial scaffold)
- New Vercel project: `hgpg-team-dash` on team `team_FietQPKCmnyioG2n0FdteQCV`
- Live at: **`team.homegrownpropertygroup.com`** (CNAME via GoDaddy → cname.vercel-dns.com)
- Tab 1 (Inventory + Showings) end-to-end working:
  - Grid filtered by office key `CAR118249224`
  - Status filter pills (Live / Closed 90 / All / Canceled+Expired)
  - Sync freshness banner reading from `mls_sync_state`
  - `/inventory/[listingId]` drilldown with showings, ListTrac platform views, key dates
  - Banner when listing isn't onboarded into the Listing Report Portal
- Magic link auth + email allow-list via `TEAM_EMAILS` env var (Brian, Ashley, Brenda, Taylor, Closings)
- Brand-correct UI: Cooper Hewitt body, Sansita display, navy/steel/light-steel/off-white tokens, status pills (signature brown for Under Contract, navy for Active, etc.)
- Spec at `projects/team-dashboard.md` (commit `f1582a1`)

### Critical fixes shipped during deploy
- **Supabase RLS policies added** (migration `team_dash_read_mls_tables`): `mls_property`, `mls_member`, `mls_sync_state` had RLS on but zero policies, so anon key returned empty result sets. Added authenticated-role SELECT policies on all three.
- **Performance index added** (migration `team_dash_office_key_index`): MLS Grid table has 2.57M rows. No index on `list_office_key` was causing 8-second statement timeouts on the `?status=all` filter. Added composite `(list_office_key, standard_status, list_date DESC)` and `(listing_id)` indexes. Query time went from timing-out to ~36ms.

### Supabase auth config
- Site URL: kept as `https://listing-report-deploy.vercel.app` (do NOT change — Listing Report Portal uses Google OAuth and may rely on it; Brian found that callback URL was already in redirect list)
- Redirect URLs added:
  - `https://team.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
- Existing Listing Report Portal redirect URLs left intact

### Key data discoveries
- **Office key** is `CAR118249224` — NOT R04075 (that's a Canopy display ID, doesn't appear in MLS Grid feed)
- 63 total team listings: 3 Active, 1 Active Under Contract, 1 Pending, 42 Closed, 14 Canceled, 2 Expired
- Active+UC+Pending: 1444 Society Hill Rd, 7807 Lamington Dr (Pending), 11230 Mallard Crossing Dr, 13002 Butters Way (UC), 00 Stratton Farm Rd
- 4 distinct list_agent_keys: Brian (CAR39987928), Ashley (CAR25452179), Brenda (CAR26251536), Taylor (CAR266730170)
- Vessie Lopez and Lauren McCarron are on the office key but haven't listed yet
- Address composition required — `unparsed_address` is null on most rows, build from `street_number + street_name + street_suffix`
- mls_property `listing_id` has `CAR` prefix; `listings.mls_number` does not. Strip on join: `REPLACE(listing_id, 'CAR', '')`
- MLS Grid Media table exists but zero rows synced. Hero photos fall back to portal `listings.hero_photo_url` then placeholder

### Pickup notes for next session

**Goal: Build Tab 2 (Market Intelligence)**

Spec stub from `projects/team-dashboard.md`:
- Search bar: zip / subdivision / radius from address
- Recharts: median price (12mo), DOM, sold volume, list-to-sold ratio, months-of-supply
- Active inventory count + 30-day movers (new pendings, new closings, price changes)
- All charts read from `mls_property` filtered by area
- Reuse Recharts components from TM where possible

Implementation order suggestion:
1. Decide search input UX (zip-first is simplest, subdivision/address adds complexity)
2. Build the area-filter SQL helpers (separate from inventory.ts — new file `lib/market.ts`)
3. Build the snapshot page with stats cards
4. Add Recharts time-series sections one chart at a time
5. Add 30-day movers list at bottom

**Open follow-ups specific to team dashboard**
- Visual polish only after Tab 2 ships (no point iterating on shell now)
- MLS Grid Media sync still parked — until it runs, hero photos only appear for ~7 portal-onboarded listings
- Consider adding a "non-team listings I have a stake in" feature later (buyer-side deals) — not in current spec

---

## Locked queue

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done
2. 🔄 Team Dashboard Tab 2 (Market Intelligence) — NEXT SESSION
3. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr)
4. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
5. 🤖 FUB AI Agent rebuild — gated on Sendblue eval

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard now covers the team-facing need)

---

## Prior session: 2026-05-08 (AM) — Strategic queue refresh + brain hygiene 🟢

### What got done
- Verified Pixel/CAPI rollout status across all consumer sites (Buyers, Sellers, NewCon shipped; Signature pending; TM/marketinganalyzer intentionally skipped)
- Pulled forward the locked queue from the 2026-05-06 strategic review

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase
- Supabase project renames for hygiene completed

### Deferred / Phase 2 for brain-app
- iPhone smoke test, Cooper Hewitt self-hosted, file rename/delete, diff view, cross-file search
