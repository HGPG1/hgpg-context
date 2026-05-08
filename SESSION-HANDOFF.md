<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-08 (PM2) — Team Dashboard Tab 2 SHIPPED 🟢

### What got shipped
- Tab 2 (Market Intelligence) live at `team.homegrownpropertygroup.com/market`
- Three search modes: zip / subdivision typeahead / address radius
- 6-month trend charts (median price, DOM, sold volume) via Recharts
- Snapshot stats: active, pending+UC, closed 30d, months of supply, list-to-sold ratio, 30-day movers
- Side-by-side area comparison (brown comparison line on charts)
- Save area to `team_watched_areas` (per-user, RLS-gated)
- Watched areas chip strip on the page

### Repo / deploy state
- Repo: `HGPG1/hgpg-team-dash` on `main`
- Last commit pushed: fix2 for compare button (URL params + show second search bar after click)
- Vercel project: `prj_Q4dFcHGUvmUbUPaDydcHhdS2Laxd`
- Live: `team.homegrownpropertygroup.com`

### Database changes applied this session
- `team_watched_areas` table with RLS (user-scoped read/insert/delete)
- `search_subdivisions(text)` RPC function (security definer, returns canonical+active+lifetime)
- All Tab 1 fixes still in place: RLS policies on mls_property/mls_member/mls_sync_state, composite index on (list_office_key, standard_status, list_date DESC), index on listing_id

### Critical performance/data discoveries
- **`mlg_can_view = true` filter** is required to hit existing partial indexes (postal_code, comp_search). Without it queries seq-scan 2.57M rows and timeout. Added to all market queries.
- **PostgREST default 1000-row cap** hit on busy zips like 28277 (1348 recent rows). Bumped query ranges to 10,000.
- **Geocoding via Nominatim/OSM** (free, fair-use compliant) for radius mode. Cached server-side.
- **Bounding box + haversine refinement** for radius search — bbox query hits Postgres indexes, then JS filters down to true radius distance.

### PAT scope expansion (this session)
- Brain App PAT (used for `/api/external/write` to push to hgpg-context) was scoped to hgpg-context contents:write only
- **Expanded to "All repositories" with Contents: Read and write** under HGPG1 org
- This means future code pushes to ANY HGPG1 repo can be done via the Brain App write API instead of tarball-and-paste flow
- Brian updated the env var in Brain App Vercel project; redeploy may have been needed

### Pickup notes for next session

**Test Tab 2 once Brian is back:**

1. Hard refresh `team.homegrownpropertygroup.com/market`
2. Verify zip search renders snapshot for `28277`
3. Subdivision typeahead works (try `Sun`, `Stratton`)
4. Address radius works (try `5022 Cressingham Dr Indian Land SC` with 1 mi)
5. Click "+ Compare to another area" — should reveal brown-bordered "Pick a comparison area" card with a second search bar
6. Fill in second area → both panels render side-by-side
7. Save area → ★ Save area button → check chip strip persists on refresh

**If anything breaks:** check `Vercel:get_runtime_logs` for `hgpg-team-dash` project, `team_FietQPKCmnyioG2n0FdteQCV` team.

**Verify the PAT swap took effect:**
- After redeploy, test pushing a small change via the write API to confirm the broader PAT works on a non-hgpg-context repo
- If it failed silently, double-check env var name and value in Brain App Vercel settings

---

## Locked queue

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done
2. ⏳ Team Dashboard test pass (above)
3. 🎯 Lightweight team listing photo sync (~30 min) — see `projects/team-photo-sync.md` (commit 4c28366) — on-demand sync for ~5 active team listings, rehosted to Supabase Storage, MLS Grid TOS compliant
4. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr)
5. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
6. 🤖 FUB AI Agent rebuild — gated on Sendblue eval

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (now superseded by team-photo-sync.md for the team-only need; full sync still parked for comp photos in Tab 2 of Team Dashboard)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard now covers the team-facing need)

---

## Prior session: 2026-05-08 (PM) — Team Dashboard Tab 1 SHIPPED 🟢

### What got shipped
- New repo: `HGPG1/hgpg-team-dash` (commit `e9d1366` initial)
- New Vercel project: `hgpg-team-dash`
- Tab 1 (Inventory + Showings) end-to-end working
- Magic link auth + email allow-list via `TEAM_EMAILS`

### Critical fixes shipped during deploy
- Supabase RLS policies added (migration `team_dash_read_mls_tables`)
- Performance index added (migration `team_dash_office_key_index`)
- Office key is `CAR118249224` (NOT R04075)

### Supabase auth config
- Site URL kept as `https://listing-report-deploy.vercel.app` (Listing Report Portal Google OAuth may rely on it)
- Redirect URLs added: `https://team.homegrownpropertygroup.com/**`, `http://localhost:3000/**`

### Key data discoveries
- 63 total team listings: 3 Active, 1 UC, 1 Pending, 42 Closed, 14 Canceled, 2 Expired
- Active+UC+Pending: Society Hill Rd, Lamington Dr (Pending), Mallard Crossing Dr, Butters Way (UC), Stratton Farm Rd
- Address composition required (street_number + street_name + street_suffix)
- mls_property listing_id has `CAR` prefix; listings.mls_number does not — strip on join

---

## Prior session: 2026-05-08 (AM) — Strategic queue refresh + brain hygiene 🟢

- Verified Pixel/CAPI rollout status across all consumer sites (Buyers, Sellers, NewCon shipped; Signature pending; TM/marketinganalyzer intentionally skipped)
- Pulled forward the locked queue from the 2026-05-06 strategic review

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

- New repo: `HGPG1/brain-app`, live at `brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Resend custom SMTP wired into `HGPG Core` Supabase
- Supabase project renames for hygiene completed
