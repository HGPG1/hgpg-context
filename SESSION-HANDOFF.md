<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Last session: 2026-05-08 (PM2) — Team Dashboard Tab 2 SHIPPED & VERIFIED 🟢

### What got shipped today (full day)
**Morning:**
- Verified Pixel/CAPI rollout status (Buyers/Sellers/NewCon done; Signature pending; TM/marketinganalyzer intentionally skipped)
- Pulled forward locked queue from 2026-05-06 strategic review

**Afternoon:**
- Team Dashboard Tab 1 (Inventory + Showings) — live at `team.homegrownpropertygroup.com`
- Team Dashboard Tab 2 (Market Intelligence) — three search modes, 6mo trend charts, comparison, saved areas

### Tab 2 verification (2026-05-08 PM2)
- ✅ Zip search renders snapshot
- ✅ Subdivision typeahead working
- ✅ Address radius geocodes and renders
- ✅ Compare to another area reveals second search bar; both panels render side-by-side
- ✅ Save area persists across sessions

### Repo / deploy state
- Repo: `HGPG1/hgpg-team-dash` on `main`
- Vercel project: `prj_Q4dFcHGUvmUbUPaDydcHhdS2Laxd`
- Live: `team.homegrownpropertygroup.com`

### Database changes applied this session
- `team_dash_read_mls_tables` — RLS SELECT policies on mls_property, mls_member, mls_sync_state
- `team_dash_office_key_index` — composite index on (list_office_key, standard_status, list_date DESC) + listing_id index
- `team_dash_watched_areas` — table with user-scoped RLS
- `team_dash_subdivision_search_rpc` — `search_subdivisions(text)` security-definer function

### Critical performance/data discoveries
- `mlg_can_view = true` filter required on all Tab 2 queries to hit existing partial indexes (without it: 18s+ timeouts; with it: <300ms)
- PostgREST default 1000-row cap silently truncates; bumped to `range(0, 9999)` on busy zips
- CAR-prefix strip required on listing_id to join MLS Grid → portal listings
- Nominatim geocoding works fine with proper user-agent header
- Address composition required (street_number + street_name + street_suffix) — `unparsed_address` is null on most rows

### PAT scope expansion (this session)
- Brain App PAT expanded from `hgpg-context` only → "All repositories" on HGPG1 org with Contents: Read and write
- Future code pushes to ANY HGPG1 repo can use the Brain App write API instead of tarball flow
- env var updated in Brain App Vercel project, redeployed

---

## Locked queue

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done (Path A organic, Path B Meta-bypass, register ScoreCompleted as Custom Conversion, wire Lead Ads, remove META_TEST_EVENT_CODE)
2. 🎯 Lightweight team listing photo sync (~30 min) — see `projects/team-photo-sync.md` (commit `4c28366`). On-demand for ~5 active team listings, rehosted to Supabase Storage, MLS Grid TOS compliant
3. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr)
4. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
5. 🤖 FUB AI Agent rebuild — gated on Sendblue eval (signup, FUB integration test, sales demo, LoopMessage 2nd-sender quote)

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media full sync (parked — superseded by team-photo-sync.md for team-only need; full sync still required for Tab 2 comp photos if/when wanted)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard now covers the team-facing need)

---

## Recommended next session pickup

Two paths depending on appetite:

**Path A (short session, high satisfaction):** Sellers guide Pixel QA. Closes a ✅ on the queue. ~30 min.

**Path B (medium session, visible team value):** Lightweight team listing photo sync. Adds hero photos to the 5 active Tab 1 cards. ~30 min build per design note. Highly visible polish.

**Path C (long session, growth-focused):** Start Buyer Alerts. ~4-5 hr. Highest deal-velocity lever.

My recommendation if usage feedback from Ashley/Brenda/Taylor on Team Dashboard arrives first: Path B (photos), then iterate Tab 2 based on what they actually use, then Buyer Alerts.

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
- mls_property listing_id has `CAR` prefix; listings.mls_number does not — strip on join

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

- Repo: `HGPG1/brain-app`, live at `brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Resend custom SMTP wired into `HGPG Core` Supabase
- Supabase project renames for hygiene completed
