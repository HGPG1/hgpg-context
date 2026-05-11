<!-- Last Updated: 2026-05-11 -->

# Sellers Guide

**Status:** 🟢 SHIPPED, ad-ready pending QA pass

## URLs

- Production: https://sellersguide.homegrownpropertygroup.com
- Repo: `HGPG1/charlotte-sellers-guide-vercel` (private), branch `main`
- Vercel project: `prj_2vnp0o6qfBdjNWaZsagIyUugEnrN` (team `team_FietQPKCmnyioG2n0FdteQCV`)
- Output dir: `public/public/public/`

## Stack

Static HTML across 7 pages, Vercel-hosted, serverless API routes for:

- `/api/assessment/submit` — Home Grown Selling Score v2 lead capture (single-page, end-of-flow)
- `/api/assessment/create` + `/submit-ratings` — legacy v1 wizard flow, kept for backward compat
- `/api/fub-lead` — generic lead intake (Contact an Agent modal, etc.) → FUB Events API
- `/api/meta/capi.js` — server-side Conversions API mirror, ESM, shared `event_id` dedup

## Lead capture

**Database:** Supabase project `fkxgdqfnowskflgbuxhm` (HGPG Signature + Relocation) — note this is NOT in HGPG Core.

Tables:

- `seller_assessments` — completed score submissions
- `seller_assessment_ratings` — per-item ratings (v1 wizard)
- `seller_assessments_v2_summary` — v2 5-page flow summary
- `seller_verification_codes` — 6-digit email verify codes (non-Meta traffic only)

**FUB ingestion:**

- All leads forwarded via `/api/fub-lead` using Events API (not Contacts API)
- Tags applied: `sellers-guide-2026`, `website-lead`, plus source-specific (`home-score-contact`, etc.)
- UTM custom fields use `customXXX` API names not labels (fixed 2026-05-04)

## Meta Pixel + CAPI

- Pixel ID: `861295553661596` (HGPG - Sellers Guide)
- Browser pixel injected on all 7 HTML pages
- Server CAPI: `api/meta/capi.js`, ESM, shared `event_id` for dedup
- Events firing: `AssessmentStarted`, `Lead`, `ScoreCompleted`, `QuizStarted`, `QuizCompleted`
- Standard events match Meta requirements; custom events for funnel analysis

## Paid traffic configuration

- `utm_source=meta` bypasses 6-digit email verify gate
  - Meta-traffic flow: name + email + phone → straight through, no verify code
  - Organic flow: still requires email verification before lead saves
  - Implemented in both client (UI skip) and `api/assessment/create.js` server-side guard
- UTM params forwarded to FUB custom fields for attribution

## Home Grown Selling Score v2

- Replaced v1 46-item wizard on 2026-05-07
- 5 pages × 4 items = 20 questions
- Internal scoring 4/2/1/-1, raw range -20 to 80
- Display normalized 0 to 80 via smooth curve (raw -20..80 → display 0..80)
- 80 cap is intentional — "Market Ready" 85+ tier unreachable by design
- Tier labels: Market Ready / Strong Foundation / Solid Bones / Pre-Market Project
- Lead capture at END of flow (no re-entry, no 2nd verification step)
- NeverBounce email validation wired in, re-validates on edit

## Recent build history (most recent first)

- 2026-05-07 — Clear inline display on setNbStatus so class rules win after idle
- 2026-05-07 — Re-validate email on edit + drop Category Breakdown from results
- 2026-05-07 — Wire NeverBounce email validation into Home Grown Selling Score v2
- 2026-05-07 — Rename Home Selling Score → Home Grown Selling Score, cap curve at 80
- 2026-05-07 — Home Selling Score v2: 5 categories, 20 items, single-page lead capture
- 2026-05-07 — Harden .gitignore (.vercel, .env*.local)
- 2026-05-06 — Fix UTM custom field API names (customXXX not labels)
- 2026-05-06 — CAPI converted to ESM (import/export default for type:module)
- 2026-05-05 — Meta Pixel + CAPI shipped, UTM bypass, FUB attribution
- 2026-05-04 — Schema enrichment + restored home-selling-score schema
- 2026-05-02 — Header/footer matched to buyers guide pattern, Contact-an-Agent modal

## Ready for ads — QA checklist before spend

Everything is wired. Before turning on ad spend:

- [ ] Meta Events Manager: confirm `Lead` events appearing from browser AND server, dedup working post-ESM-fix
- [ ] End-to-end test from `?utm_source=meta` URL — verify bypass works, lead lands in FUB with correct tags + UTM custom fields populated
- [ ] FUB Automations 2.0: confirm `sellers-guide-2026` tag triggers desired drip / agent assignment
- [ ] Spot-check NeverBounce rejection path on a known-bad email
- [ ] Confirm Contact-an-Agent modal lead also lands cleanly (separate code path from score submission)

## Known notes

- "Phase 1 ads test markers" left in code from 2026-05-05 CAPI commit — flag for cleanup post-launch
- Database lives in Signature/Relocation Supabase, not HGPG Core. If we ever consolidate, this needs to come along.
- 4 domains aliased on Vercel: `sellersguide.homegrownpropertygroup.com` is canonical; others are Vercel defaults + git-branch aliases
