<!-- Last Updated: 2026-05-12 -->

# New Construction Site - Phone Capture Build

**Status:** 🟡 SPEC LOCKED, READY TO SHIP
**Repo:** `HGPG1/charlotte-new-construction-nextjs` (branch: `main`)
**Domain:** `newconstruction.homegrownpropertygroup.com`
**Driver:** Capitalize on imminent paid ad spend by capturing phone alongside email across all 4 lead forms.

## Strategy decisions (locked 2026-05-12)

| Decision | Choice |
|---|---|
| Phone strategy across the 4 forms | **Tiered** - optional on Guide/Quiz/Calculator, required on Builder Intro |
| Copy nudge style | **Benefit-led**, tailored per form (see table below) |
| FUB destination | **Standard `mobile` phone on contact** - no custom field |
| SMS speed-to-lead automation | **Deferred** - ship phone capture first, verify, then SMS PR as follow-up |

## Per-form copy

| Form | FUB Template ID | Required? | Label | Helper text |
|---|---|---|---|---|
| Guide Delivery | 1147 | optional | `Phone (optional)` | `We'll text you the guide link too` |
| Quiz Complete | 1150 | optional | `Phone (optional)` | `We'll text you your community matches` |
| Calculator | 1149 | optional | `Phone (optional)` | `We'll text you the breakdown` |
| Builder Intro | 1151 | **required** | `Phone *` | `The builder rep will reach out within 24 hours` |

## Technical changes

### 1. `src/lib/fub.ts`
Add `normalizePhone()` helper - strip non-digits, prepend `+1` if 10 digits, accept `+1` if 11 digits starting with 1, return `null` for anything else (junk like "123" rejected, never passed to FUB). Update the FUB Events payload builder so `person` conditionally includes `phones: [{value, type: 'mobile'}]` when normalized phone is non-null.

### 2. Meta CAPI hashing
Locate the file building Meta CAPI `user_data` (probably `src/lib/meta.ts` or inline in API routes). Add SHA-256 hashed `ph` parameter when normalized phone present. Hash digits-only (NO leading `+`, NO formatting - Meta's requirement). Verify `em` hash uses `sha256(email.trim().toLowerCase())`.

### 3. The 4 form components
- Input attrs: `type="tel"`, `inputMode="tel"`, `autoComplete="tel"`, `name="phone"`, `placeholder="(704) 555-1234"`
- Match existing email field styling, don't reinvent
- Client-side validation: if any non-whitespace content AND `normalizePhone` returns null, inline error "Please enter a valid US phone number", block submit
- Builder Intro only: block submit if empty/whitespace, error "Phone number is required for builder introductions"
- No em dashes per brand rules

### 4. The 4 API routes (`src/app/api/<name>/route.ts`)
- Destructure `phone` with `null` default
- Normalize once at top via `normalizePhone()`
- Pass into both FUB call + Meta CAPI fire
- Builder Intro route only: return 400 if `normalizedPhone` is null

## Smoke test plan

Run against Vercel preview URL before merging:

1. Guide Delivery, no phone → 200, FUB contact created email-only
2. Quiz, phone `(704) 555-1234` → 200, FUB contact has `+17045551234` as mobile
3. Calculator, phone `12345` → inline error, submit blocked
4. Builder Intro, empty phone → inline error, submit blocked
5. Builder Intro, valid phone → 200, FUB contact + Meta Test Events both fire
6. Meta Events Manager → Test Events → confirm `ph` parameter on Lead event payloads from steps 2 and 5

All 5 pass → merge to main, Vercel auto-deploys prod.

## Claude Code prompt (paste-ready)

The full prompt was generated and handed to Brian in conversation 2026-05-12. Stored verbatim in `SESSION-HANDOFF.md` for pickup convenience.

## Follow-up (parked)

**SMS speed-to-lead automation on Builder Intro:** After phone capture is live and a real lead has been captured with phone attached, build FUB Automation 2.0 rule that triggers a 90-second SMS auto-response on Builder Intro submissions. Separate PR, separate session.

## References

- Original conversation: 2026-04-30 (Meta Pixel + CAPI install on this site)
- Phone capture decision conversation: 2026-05-12
- Domain Pixel ID: 1880396459290092 (Meta Pixel + CAPI live, all 6 funnel events firing)
- API routes share `src/lib/fub.ts`
- FUB API key already in Vercel env from prior build
