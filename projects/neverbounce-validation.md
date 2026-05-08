<!-- Last Updated: 2026-05-08 (originally specced 2026-05-05) -->

# NeverBounce Email Validation

- **Status:** 🔵 Spec written, not built
- **Live URL (target):** newconstruction.homegrownpropertygroup.com/incentives
- **Repo:** HGPG1/charlotte-new-construction-nextjs
- **Stack:** Next.js, Vercel functions

## Goal

Real-time email validation on the `/incentives` form to prevent invalid or typo'd email addresses from reaching FUB. Today, common typo cases like `johnsmith@gmail.como` slip through and pollute downstream automation, email deliverability stats, and lead-scoring signals.

## Requirements

1. Integrate NeverBounce real-time validation API (`/single/check` endpoint)
2. Validation fires on blur of the email input field, debounced 500ms
3. Inline feedback under the field:
   - Green check for valid
   - Red X for invalid
   - Yellow warning for risky (catchall, disposable, unknown)
4. Hard-block form submission on `invalid` result
5. Allow submission on `risky` result, but show warning text
6. Store NeverBounce result code in form metadata sent to FUB (custom field) for later analysis
7. API key in Vercel env vars, never client-side
8. Validation runs server-side via a Vercel function — do NOT call NeverBounce from the browser

## Architecture

### Vercel function

New route at `/api/validate-email`:

- Accepts: `{ email: string }`
- Calls NeverBounce `/single/check` with the API key from env
- Returns: `{ result: "valid" | "invalid" | "disposable" | "catchall" | "unknown", flags: string[], suggested_correction?: string }`
- Caches by email for 24 hours (cheap memory map or Vercel KV) to avoid double-billing on form re-validation

### Client-side

500ms debounced blur handler on the email input:

- Fetch `/api/validate-email`
- Render inline status with color-coded feedback
- Set form-state `emailValidationStatus` so the submit handler can gate

On submit:

- Re-validate if not already validated (prevents stale state)
- Block if `result === "invalid"`, show error
- Allow with warning if `result` is `disposable` / `unknown` / `catchall`
- Submit with `neverbounce_result` field appended to FUB payload

### FUB integration

Append `neverbounce_result` to the existing `/incentives` form FUB payload. Field needs to exist in FUB before this ships or the API drops it silently.

## Pre-build checklist

- [ ] NeverBounce account created
- [ ] API key in hand, added to Vercel as `NEVERBOUNCE_API_KEY`
- [ ] FUB custom field created — recommend name `email_validation_status` (text type, on People)
- [ ] Confirm whether existing `/incentives` form has any other validation that would conflict

## Phase 2 (deferred until 50+ leads through funnel)

- Domain blocklist for industry/competitor emails (Stephen Cooley RE, KW, Compass, eXp, Realty One Group, etc.)
- Decision needed: hard block vs. flag-and-allow for industry leads
- Recommend flag-and-allow with a "industry_email" tag in FUB so they're routed to a separate sequence or excluded from automation
- Research at 50-lead mark: what % were flagged risky? Did any convert? That informs the hard-block decision.

## Cost estimate

- Setup: 2-4 hours dev time
- NeverBounce API: ~$1-15/month depending on volume (their free tier covers 1k validations/month)
- No recurring infrastructure cost (runs on existing Vercel)

## Pickup notes

- This is a small, self-contained build — single PR, single migration if any (likely none, just the FUB field add)
- The Vercel function is the security boundary. Keep the NeverBounce key server-only.
- If we ever expand beyond `/incentives`, the same `/api/validate-email` route can be reused on any other form (Sellers Guide, sellersguide score, etc.) without changes.
