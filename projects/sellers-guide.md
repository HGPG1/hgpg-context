<!-- Last Updated: 2026-05-08 -->

# Sellers Guide

- **Status:** 🟢 Live
- **URL:** sellersguide.homegrownpropertygroup.com
- **Repo:** HGPG1/charlotte-sellers-guide-vercel
- **Vercel team:** `team_FietQPKCmnyioG2n0FdteQCV`
- **Output dir:** `public/public/public/` (legacy nested path, do not change)
- **Supabase:** `fkxgdqfnowskflgbuxhm` (HGPG Signature + Relocation), `seller_assessments` + `seller_assessment_ratings` tables
- **Branding:** rebrand complete, uses HGPG navy/steel palette, Cooper Hewitt + Sansita fonts, no green

## What lives in this project

The Sellers Guide is a multi-section static site that funnels home sellers into HGPG's pipeline. Built on Vercel with a few server-side API routes for lead capture and Meta Pixel CAPI relay.

### Sections

| Section | Path | Notes |
|---|---|---|
| Home / landing | `/` | Hero + section nav |
| Home Grown Selling Score | `/home-selling-score/` | Wizard with lead capture (see "Home Grown Selling Score v2" below) |
| Equity calculator | `/equity-calculator/` | Estimate net proceeds at sale |
| Pre-list checklist | `/pre-list-checklist/` | Static reference content |
| Marketing analyzer | `/marketing-analyzer/` | Compare HGPG marketing vs typical agent |
| Signature page | `/signature/` | Lead capture for the HGPG Signature service |
| Other static pages | various | Process explainers, FAQ, etc. |

### API routes

| Endpoint | What it does |
|---|---|
| `/api/fub-lead` | Canonical FUB push for ALL Sellers Guide lead capture. Handles UTM custom field mapping, tagging, Meta CAPI bypass tag for `?utm_source=meta` traffic. Other endpoints forward here. |
| `/api/meta/capi` | Server-side CAPI relay for Meta Pixel events. Mirrors browser pixel events with `event_id` for dedup. |
| `/api/assessment/submit` | NEW (2026-05-08): single-call lead + score capture for Home Grown Selling Score v2. Writes to `seller_assessments`, forwards to `/api/fub-lead`. |
| `/api/assessment/create` | LEGACY: original two-stage assessment flow (create row → submit ratings). Kept for backward compat with cached clients. |
| `/api/assessment/submit-ratings` | LEGACY: companion to `/create` for the old per-item rating wizard. Kept for backward compat. |
| `/api/verification/send-code` | LEGACY: phone verification SMS for the original assessment flow. Not used by v2. |
| `/api/verification/verify-code` | LEGACY: phone verification check. Not used by v2. |

### Meta Pixel + CAPI

- **Browser pixel** loaded in `<head>` of every page via shared snippet
- **Server-side CAPI** mirrors via `/api/meta/capi` with `event_id` for dedup
- **7 FUB UTM custom fields** wired to capture attribution: `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `landing_page`, `referrer`
- **UTM bypass:** `?utm_source=meta` traffic gets the `meta-bypass` tag in FUB so it doesn't double-count in Meta's optimization (Meta already sees those leads)
- **Patch script:** `patch-home-selling-score.py` (in repo root) handles applying pixel anchors to the Home Grown Selling Score page idempotently. Anchor markers: `META_BYPASS`, `AssessmentStarted`, `ScoreCompleted`, `metaBypass`, `getUtmsForFub`. If you replace the score HTML, these markers MUST be preserved or the patch will fail to detect "already patched" state.

---

## Home Grown Selling Score v2 (shipped 2026-05-08)

The flagship lead-gen tool of the Sellers Guide. Replaced the original 46-item wizard.

### Score model

- **Internal raw score** (saved as `total_score` in DB): 4 / 2 / 1 / -1 points per item, range -20 to +80, NEVER shown to clients
- **Client-facing display score** (saved as `score_percentage`): normalized 0-80 via `Math.round((raw + 20) * 0.8)`. Smooth curve, no flat ceiling. Max 80 is intentional, there's always something to discuss pre-listing, and a perfect score removes the reason for a consult.
- **Tier** (saved as `tier_name`):
  - `85+` → "Market Ready", UNREACHABLE BY DESIGN (kept in code for completeness; max display = 80)
  - `70-84` → "Strong Foundation", top tier the client can hit (requires raw 68+)
  - `55-69` → "Solid Bones"
  - `0-54` → "Pre-Market Project"

### Structure

5 categories × 4 items each = 20 items total, paginated as 1 page per category:

1. **Curb Appeal** (lawn, exterior paint, front door, driveway/lighting)
2. **Kitchen** (cabinets, counters, appliances, layout/flow)
3. **Bathrooms** (primary, secondary, count vs comps, grout/caulk)
4. **Living Spaces** (flooring, paint, natural light, clutter)
5. **Exterior and Systems** (roof, HVAC, water heater/electrical/plumbing, backyard/deck/fence)

### Flow

1. Intro card with 5/20/~4min stats, single Start button
2. Wizard: 5 pages, 4 items per page, radio-select answers, Next button enables when all 4 are answered
3. Results: animated score ring, tier label, per-category breakdown bars, top 3 recommendations
4. Inline lead form (first/last/email/phone/address/timeline), submit fires `/api/assessment/submit`
5. Success state replaces form: "You're all set, plan is on its way"

Lead capture is at the END of the flow, not the beginning. No phone verification step in v2 (the legacy SMS verification flow is still in the codebase for backward compat but not used).

### Score DB schema (seller_assessments)

Original columns retained from v1 schema. Six columns added in migration `seller_assessments_v2_2026_05_07`:

- `answers_json` (jsonb), raw answers map `{ item_id: { v: number, t: string } }`
- `breakdown_json` (jsonb), per-category breakdown with pct + raw point total
- `tier_name` (text), Market Ready / Strong Foundation / Solid Bones / Pre-Market Project
- `schema_version` (text), distinguishes v1 from `v2-2026-05-07`
- `utms_json` (jsonb), full UTM payload at submit time
- `timeline` (text), seller's stated timeline (0-3mo / 3-6mo / 6-12mo / 12+mo)

Plus 2 partial indexes (`tier_name`, `schema_version`) and a reporting view: `seller_assessments_v2_summary` (filtered to v2 rows, sorted by `created_at` DESC).

Migration is purely additive. Nullable on all new columns. The legacy v1 flow continues to work unchanged.

### Pixel events fired

- `AssessmentStarted` on Start button click (CustomEvent)
- `ScoreCompleted` on results render (CustomEvent, includes value + tier)
- `Lead` on successful inline form submit (standard FB event)

All three fire client-side via `fbq()` with optional CAPI mirror via `/api/meta/capi`. Server-side relay happens through the FUB push in `/api/assessment/submit` → `/api/fub-lead`.

### Branding

- Page title: "Home Grown Selling Score" (NOT "Home Selling Score", that was the original wizard's name)
- Brand tag in header: "Home Grown Property Group"
- Score ring label: "Your Score" (not "out of 100", we don't surface the implicit cap)

---

## Pickup notes for next session

- Old API routes (`/api/assessment/create`, `/api/assessment/submit-ratings`, `/api/verification/*`) can be removed in a future cleanup once we confirm zero traffic for a week. Vercel function logs are the source of truth.
- Meta Custom Conversion registration for `ScoreCompleted` is still pending, the events fire correctly, they just need to be promoted to a Custom Conversion in Meta Events Manager so they can be used in ads optimization.
- If you replace the score HTML again, run `patch-home-selling-score.py` to verify the pixel anchors are intact. The script is idempotent, safe to re-run.
- The `OUTPUT-DIR-IS-public/public/public/` weirdness is intentional legacy. Don't try to flatten it, Vercel routing depends on this exact path.
- FUB lead `source` field for v2 score submissions reads `home-selling-score`. If you see `Sellers Guide 2026` instead, the source pass-through in `/api/assessment/submit` → `/api/fub-lead` broke.

## Verification queries

Recent v2 score submissions:

    SELECT id, contact_name, contact_email, total_score, score_percentage, tier_name, timeline, created_at
    FROM seller_assessments
    WHERE schema_version = 'v2-2026-05-07'
    ORDER BY created_at DESC LIMIT 20;

Or use the view:

    SELECT * FROM seller_assessments_v2_summary LIMIT 20;

Tier distribution check (once you have data):

    SELECT tier_name, COUNT(*) FROM seller_assessments
    WHERE schema_version = 'v2-2026-05-07'
    GROUP BY tier_name ORDER BY COUNT(*) DESC;

## Recent commits

- `3133a81` (2026-05-08), fix(score): clear inline display on setNbStatus so class rules win after idle (the bug that hid result UI after edit)
- `7d86384` (2026-05-08), fix(score): re-validate email on edit + drop Category Breakdown from results
- `dad98fb` (2026-05-08), Home Grown Selling Score: rename + curve math to cap at 80
- `7b8f27c` rebased (2026-05-07), Home Selling Score v2: 5 categories, 20 items, single-page lead capture (initial)
- `7b8f27c` (2026-05-07), Harden .gitignore: add .vercel and .env*.local

## NeverBounce email validation (live as of 2026-05-08)

The Home Grown Selling Score lead form now uses NeverBounce real-time email validation, mirroring the pattern from charlotte-new-construction-nextjs /incentives.

Implementation:
- Server-side proxy at `/api/validate-email` (ESM, fail-open philosophy, 10s AbortController timeout, format pre-check to save credits)
- Client-side: debounced 500ms validation on email blur AND on input edits (so corrections re-validate without re-tabbing)
- Inline status under email field with 5 states: idle, checking, valid (green), warning (yellow), error (red)
- Hard-block on result `invalid` or `disposable`
- Allow with warning on `catchall` or `unknown` (fail-open: a fake domain is yellow not red, but FUB still gets the raw verdict for downstream filtering)
- Spelling-mistake softener: typo with no suggested correction shows yellow not red, so we don't lose real users who fat-fingered
- "Did you mean X?" suggestion button when NeverBounce returns one
- Results persist to Supabase `seller_assessments.email_validation_status` + `email_validation_flags` (migration `seller_assessments_add_email_validation_2026_05_08` added the columns)
- FUB receives `email_validation_status` and `email_validation_flags` via the existing `/api/fub-lead` route, mapping to `customEmailValidationStatus` field (id 149) and auto-tags `Email-Invalid` / `Email-Disposable`

Env var: `NEVERBOUNCE_API_KEY` in Vercel project (Sensitive, Production + Preview only). Same key as the new construction project.

Lessons learned debugging this build:
- The first deploy had a sticky-hide bug: `setNbStatus("idle")` set inline `display: none`, which then prevented later state transitions (valid/warning/error) from showing. Fix was to clear the inline display before re-applying state. Pattern to remember: prefer class-based visibility over inline style toggles.
- The Category Breakdown on the results page showed all categories at 100% even when score was 80 (because the curve caps at 80 by design). Visually contradictory, dropped entirely. Score + tier + copy carry the message.
