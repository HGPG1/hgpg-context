<!-- Last Updated: 2026-05-11 -->

# Sellers Guide

**Status:** 🟢 SHIPPED + AD-READY. Outstanding: FUB Automation 2.0 on `sellers-guide-2026` tag (not yet built).

## URLs

- Production: https://sellersguide.homegrownpropertygroup.com
- Repo: `HGPG1/charlotte-sellers-guide-vercel` (private), branch `main`
- Vercel project: `prj_2vnp0o6qfBdjNWaZsagIyUugEnrN` (team `team_FietQPKCmnyioG2n0FdteQCV`)
- Output dir: `public/public/public/`
- Local clone: `~/Documents/charlotte-sellers-guide-vercel` on Mac mini

## Stack

Static HTML across 7 pages, Vercel-hosted, serverless API routes for:

- `/api/assessment/submit` — Home Grown Selling Score v2 lead capture (single-page, end-of-flow)
- `/api/assessment/create` + `/submit-ratings` — legacy v1 wizard flow, kept for backward compat
- `/api/fub-lead` — generic lead intake (Contact an Agent modal, etc.) → FUB Events API
- `/api/meta/capi.js` — server-side Conversions API mirror, ESM, shared `event_id` dedup
- `/api/validate-email` — NeverBounce wrapper

## Lead capture

**Database:** Supabase project `fkxgdqfnowskflgbuxhm` (HGPG Signature + Relocation) — note this is NOT in HGPG Core.

Tables:

- `seller_assessments` — completed score submissions
- `seller_assessment_ratings` — per-item ratings (v1 wizard)
- `seller_assessments_v2_summary` — v2 5-page flow summary
- `seller_verification_codes` — 6-digit email verify codes (non-Meta traffic only)

**FUB ingestion:**

- All leads forwarded via `/api/fub-lead` using FUB Events API (POST /v1/events, not /v1/people)
- Returns full FUB person object; `/api/assessment/submit` persists `person.id` as `seller_assessments.fub_event_id` for cross-system lookup
- Tags applied: `sellers-guide-2026`, `website-lead`, source name (e.g. `home-selling-score`), plus `meta-bypass` for paid traffic and `utm:{source}` per attribution
- UTM custom fields use `customXXX` API names not labels (e.g. `customUTMSource`, `customFacebookClickID`)

## Meta Pixel + CAPI

- Pixel ID: `861295553661596` (HGPG - Sellers Guide)
- Browser pixel injected on all 7 HTML pages (including `/home-selling-score/` — was missing until 2026-05-11 patch)
- Server CAPI: `api/meta/capi.js`, ESM, shared `event_id` for dedup
- Events firing: `AssessmentStarted`, `ScoreCompleted`, `QuizStarted`, `QuizCompleted`, `Lead`
- Standard `Lead` event uses Meta standard naming so it counts as a Lead in Ads Manager; others are `trackCustom`
- All event firing on `/home-selling-score/` delegates through `window.hgpgTrack(name, params)` which handles browser fbq + CAPI POST with shared `event_id`

## Paid traffic configuration

- `utm_source=meta` bypasses 6-digit email verify gate
  - Meta-traffic flow: name + email + phone → straight through, no verify code
  - Organic flow: still requires email verification before lead saves
  - Implemented in both client (UI skip) and `api/assessment/create.js` server-side guard
- UTM params (`utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `fbclid`, `gclid`) forwarded to FUB custom fields for attribution

## Home Grown Selling Score v2

- Replaced v1 46-item wizard on 2026-05-07
- 5 pages × 4 items = 20 questions
- Internal scoring 4/2/1/-1, raw range -20 to 80
- Display normalized via smooth curve, effective cap at 80
- 80 cap is intentional — "Market Ready" 85+ tier unreachable by design
- Tier labels: Market Ready / Strong Foundation / Solid Bones / Pre-Market Project
- Lead capture at END of flow (no re-entry, no 2nd verification step)
- NeverBounce email validation wired in, re-validates on edit
- Stores answers in `answers_json` blob + breakdown in `breakdown_json` (not `seller_assessment_ratings`)

## Recent build history (most recent first)

- 2026-05-11 — `feat(fub-lead)`: return FUB person object; submit.js persists `fub_person_id`
- 2026-05-11 — `fix(assessment/submit)`: await FUB forward + writeback fub_event_id (was fire-and-forget killed by lambda teardown)
- 2026-05-11 — `fix(score)`: add missing Pixel block + delegate fireFbqEvent to hgpgTrack for browser+CAPI dedup
- 2026-05-07 — Home Selling Score v2: 5 categories, 20 items, single-page lead capture
- 2026-05-07 — Wire NeverBounce email validation
- 2026-05-07 — Rename Home Selling Score → Home Grown Selling Score, cap curve at 80
- 2026-05-06 — Fix UTM custom field API names (customXXX not labels)
- 2026-05-06 — CAPI converted to ESM (import/export default for type:module)
- 2026-05-05 — Meta Pixel + CAPI shipped, UTM bypass, FUB attribution
- 2026-05-04 — Schema enrichment + restored home-selling-score schema
- 2026-05-02 — Header/footer matched to buyers guide pattern, Contact-an-Agent modal

## QA verified 2026-05-11

End-to-end test from `?utm_source=meta&utm_campaign=preflight-final-20260511`:

- ✅ Pixel `Lead` event fired browser-side with `event_id` (verified in Meta Test Events)
- ✅ CAPI POST to `/api/meta/capi` returned 200 at same timestamps as browser events (Vercel logs)
- ✅ Shared `event_id` between browser + server = Meta dedup working (single row per event in Test Events = dedup confirmed)
- ✅ Row landed in `seller_assessments` with `schema_version=v2-2026-05-07`, `tier_name`, `email_validation_status`, all 4 UTMs in `utms_json`
- ✅ `fub_event_id` populated with FUB person ID (round-trip writeback working)
- ✅ FUB person created with correct tags: `sellers-guide-2026`, `website-lead`, `home-selling-score`, `meta-bypass`, `utm:meta`
- ✅ Stage = Lead, Source = "Sellers Guide 2026", Assigned to Brian
- ✅ NeverBounce ran and persisted `email_validation_status` + flags
- ✅ Meta-bypass flow: no 6-digit email verify step shown for utm_source=meta traffic

## Outstanding before scaling spend

- 🟡 **FUB Automation 2.0 not built yet** on `sellers-guide-2026` tag. Leads land in FUB tagged correctly but no automatic drip/assignment fires. Manual follow-up required until Automation is configured. Acceptable for initial ad launch (want eyes on first leads anyway) but blocks scaling.

## Known notes

- "Phase 1 ads test markers" left in code from 2026-05-05 CAPI commit — flag for cleanup post-launch
- Database lives in Signature/Relocation Supabase, not HGPG Core
- `/process/` and `/pricing/` are 404 — if nav links point there, dead ends. Audit nav before ad spend if copy references those pages.
- `fub_event_id` column name is legacy; value is the FUB person ID (FUB Events API returns the person, not a separate event entity, because FUB dedups on email)
- 4 domains aliased on Vercel: `sellersguide.homegrownpropertygroup.com` is canonical
- Test data from 2026-05-11 QA cleaned out of `seller_assessments`; FUB persons 31927 and 31928 deleted manually
