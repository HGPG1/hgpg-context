<!-- Last Updated: 2026-05-08 -->

# NeverBounce Email Validation

- **Status:** 🟢 Live on `/incentives` (since 2026-05-05) AND on Home Grown Selling Score v2 (since 2026-05-08)
- **Phase 2:** Rollout to remaining sites (TM, marketing analyzer, Signature, buyers guide) pending

## What it does

Real-time email validation on lead capture forms to prevent invalid or typo'd email addresses from reaching FUB. Catches common typo cases (johnsmith@gmail.como), disposable emails, role accounts, and outright bogus addresses before they pollute downstream automation, deliverability stats, and lead-scoring signals.

## Architecture

### Core endpoint

Server-side proxy at `/api/validate-email`:

- Accepts: `{ email: string }`
- Calls NeverBounce v4 `/single/check` with `address_info=1` (enables typo suggestions)
- Returns: `{ result, suggested_correction, flags }`
- 10s AbortController timeout
- **Format-checks server-side BEFORE hitting NeverBounce** — saves API credits on obviously-malformed input
- **Fail-open:** missing env var, timeout, or 5xx all return `{ result: "unknown" }` — form never breaks

### Client UX

- Debounced 500ms blur handler
- Inline status under field (green check / yellow warning / red X)
- Sequence counter to ignore stale responses if user re-edits
- Typo correction button when NeverBounce returns a `suggested_correction`
- Submit gate: hard-block on `invalid` + `disposable`, allow with warning on `catchall` + `unknown`

### Critical design pattern: UI permissive, analytics honest

The typo case (`result: "invalid"` + `flags: ["spelling_mistake"]` with no suggestion) shows a yellow warning in the UI ("looks like a typo, please double-check") but sends raw `invalid` to FUB. Display state and analytical state are kept separate:

- `displayState` drives UI (idle/checking/valid/warning/error)
- `rawResult` is what goes to FUB and Supabase

This preserves the ability to filter FUB later for typo'd emails while not blocking real humans who fat-fingered.

## Where it's live

### 🟢 newconstruction.homegrownpropertygroup.com `/incentives` (shipped 2026-05-05)

All 3 form variants (hero, card_modal, secondary). Code at `src/app/api/validate-email/route.ts` and `src/app/incentives/IncentivesClient.tsx`.

### 🟢 sellersguide.homegrownpropertygroup.com `/home-selling-score/` (shipped 2026-05-08)

Wired into the v2 lead capture form (5x4 wizard, lead capture at end). Code at `api/validate-email.js` (Vercel serverless function, ESM) and embedded JS in `public/public/public/home-selling-score/index.html`.

## Infrastructure

- **Vercel env var:** `NEVERBOUNCE_API_KEY` — Sensitive, Production + Preview only (NOT Dev). Same key shared across `charlotte-new-construction-nextjs` and `charlotte-sellers-guide-vercel` projects (set independently per project).
- **FUB custom field:** `customEmailValidationStatus` (text, label "Email Validation Status", id 149)
- **FUB auto-tags:** `Email-Invalid`, `Email-Disposable` (applied based on raw result code)
- **Supabase columns** (sellers guide only, project `fkxgdqfnowskflgbuxhm`): `seller_assessments.email_validation_status` (text), `seller_assessments.email_validation_flags` (jsonb), partial index where status IS NOT NULL. Migration `seller_assessments_add_email_validation_2026_05_08`.

## Phase 2: rollout to remaining sites

When 50+ leads have flowed through `/incentives` and the score page, revisit:

- What % of leads come through as invalid / disposable
- Whether typo rate justifies a "review and find correct contact" agent task automation
- Whether disposable rate is high enough to warrant triggering Suppress-From-Email tag automatically

Sites still pending NeverBounce wiring:
- **Transaction Manager** (TM) — staff form, lower volume but worth catching attorney typos
- **Marketing Analyzer** (`marketinganalyzer.homegrownpropertygroup.com`) — single form, low effort
- **Signature** (`signature.homegrownpropertygroup.com`) — admin endpoint
- **Buyers Guide** — when the original Manus URL question is resolved

Each site needs its own `NEVERBOUNCE_API_KEY` env var (per project), server-side route, and form-side validation UI. Pattern is reusable: copy `validate-email.js` from sellers-guide (Vercel serverless) or `validate-email/route.ts` from new construction (Next.js App Router) depending on the target's stack.

Phase 2 also covers:
- **Industry/competitor email blocklist** (Stephen Cooley RE, KW, Compass, eXp, Realty One Group). Decision needed: hard-block vs flag-and-allow. Recommendation: flag-and-allow with `industry_email` tag in FUB so they're routed to a separate sequence or excluded from automation.

## Cost

- NeverBounce API: ~$1-15/month depending on volume (free tier covers 1k validations/month)
- No recurring infrastructure cost (runs on existing Vercel functions)

## Key learnings

- **`address_info=1` query param** is required for NeverBounce to populate `suggested_correction`. Without it, you get the `spelling_mistake` flag but no fix to apply.
- **Random gibberish at @gmail.com often returns `valid`** — Gmail accepts mail at any address before bouncing. Use bogus DOMAINS for invalid testing, not bogus users at real domains.
- **FUB doesn't expose custom field API names in UI** — use `GET /v1/customFields` to find them.
- **FUB tags are created on demand** — no pre-setup needed before sending new tag strings via API.
- **Vercel "Sensitive" env vars exclude Dev environment** — local `npm run dev` cannot test; must test on Preview deploys.
- **Next.js service workers cache aggressively on Preview deploys** — requires unregister + hard reload (or incognito) to see new bundles.
- **FUB Automations 2.0 has no "Custom Field Updated" trigger** — code-side tagging is the right approach when behavior needs to fire on field-write at lead creation.

## Pickup notes

- Both sites are live and capturing validation results; let real volume build up before tuning thresholds
- The pattern is genuinely reusable across all HGPG forms — copy + paste + change the FUB integration call
- If NeverBounce credits start running low, the dashboard at `app.neverbounce.com` shows usage; auto-recharge can be enabled
