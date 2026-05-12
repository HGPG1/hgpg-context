<!-- Last Updated: 2026-05-12 -->

# New Construction Site - Phone Capture Build

**Status:** 🟢 SHIPPED 2026-05-12
**Repo:** `HGPG1/charlotte-new-construction-nextjs` (branch: `main`)
**Domain:** `newconstruction.homegrownpropertygroup.com`
**Driver:** Capitalize on imminent paid ad spend by capturing phone alongside email across all 4 lead forms.

## What actually shipped (corrected framing)

**Important reframe:** Phone was already captured and *required* on every form before this PR. This shipped as a **tiered downgrade**, not greenfield phone capture. The brain originally framed this as a new feature - that was wrong. The actual work was making phone optional on 3 forms, kept required on Builder Intro, plus added E.164 normalization, junk-input null-out, and per-form benefit-led helper copy.

**PR:** https://github.com/HGPG1/charlotte-new-construction-nextjs/pull/1
**Merge commit:** `8bca9ac` (verified)
**Prod deploy:** `dpl_2UNJu3ByZa5tbdG3WK9NDAfqdnZx` (READY, target=production)
**Author:** brian@homegrownpropertygroup.com

## Final state

| Form | Required? | Helper text |
|---|---|---|
| Guide Delivery | optional | `We'll text you the guide link too` |
| Quiz Complete | optional | `We'll text you your community matches` |
| Calculator | optional | `We'll text you the breakdown` |
| Builder Intro | required | `The builder rep will reach out within 24 hours` |

## Key implementation notes (for future sessions)

- **Only 2 form component files, not 4.** Guide/Quiz/Calculator share `src/components/LeadCaptureModal.tsx`. Builder Intro is its own `src/components/BuilderLeadModal.tsx`. Per-form copy threaded via new `phoneHelperText` prop, not a fork.
- **SMS consent checkbox is conditional on phone presence.** If no digit in the phone field, the consent checkbox is hidden entirely. This was a Claude Code judgment call (the spec was silent on it) - good instinct. Builder Intro keeps consent required since phone is required.
- **`normalizePhone()` in `src/lib/fub.ts`** is the canonical helper. Returns null on junk (e.g. `"123"`), prepends `+1` on 10-digit input, accepts `+1` on 11-digit `1` prefix. Junk is rejected with inline error, never passed to FUB.
- **Client-side mirror `normalizePhoneClient`** duplicated 5-line helper in both modal components because `src/lib/fub.ts` imports `next/server` - can't pull into `'use client'` bundle even tree-shaken.
- **FUB email now uses `type: 'home'`.** Previously had no type field. FUB matches on value either way.
- **FUB note line "Phone: not provided"** added to leads who skip phone, replacing the old "SMS Consent: NO" line so the operational state reads cleanly. Email-only leads visually distinct from phone-having-but-declined-SMS leads in FUB.
- **Branch name was `claude/phone-capture-lead-forms-Dq6ZJ`** (Claude Code harness-assigned, not `feature/phone-capture` as the original spec said). Harness convention.

## Files modified (final)

**Form components (2):**
- `src/components/LeadCaptureModal.tsx`
- `src/components/BuilderLeadModal.tsx`

**Call-site helper-text wiring (4):**
- `src/app/HomeContent.tsx`
- `src/components/FooterContactCTA.tsx`
- `src/app/quiz/QuizClient.tsx`
- `src/app/calculator/CalculatorClient.tsx`

**Lib + routes:**
- `src/lib/fub.ts` (normalizePhone + nullable phone in FubLeadInput + conditional phones[] in FUB person + conditional SMS-consent gate)
- `src/lib/meta.ts` (ph hashing was already correct; only Last Updated comment changed)
- `src/app/api/guide-lead/route.ts`, `calculator-lead/route.ts`, `quiz-lead/route.ts` (destructure phone, normalize, 400 on junk)
- `src/app/api/builder-lead/route.ts` (same + 400 if normalizedPhone null)

## Preview review (2026-05-12)

Brian ran fast-path 3-check review on preview deploy `dpl_CMzXNRubbQgUn4QdxESJSBqZgMG9`:

- ✅ Check 1: Conditional SMS consent on Guide Delivery - PASS
- ✅ Check 2: Builder Intro empty-phone block - PASS
- ⏸ Check 3: FUB record clarity - deferred (Brian not at computer; eyeball whenever next real lead comes through)

Merged on basis of passing UX-critical paths.

## Follow-up (parked - separate PR)

**SMS speed-to-lead automation on Builder Intro:** Next session. Real decision to make before building:
- **Gate the automation on the SMS consent checkbox** (legal-clean path - only fire on Builder Intro submissions where consent was checked).
- **Or fire on Builder Intro submission regardless** (faster but riskier on TCPA).

Builder Intro requires phone AND consent, so in practice both paths produce the same trigger set today. The decision matters for the audit trail and for any future form where phone might be required but consent is optional. Recommendation: gate on consent for defensibility, even if functionally equivalent right now.

## References

- Original conversation: 2026-04-30 (Meta Pixel + CAPI install on this site)
- Phone capture decision conversation: 2026-05-12
- Domain Pixel ID: 1880396459290092 (Meta Pixel + CAPI live, all 6 funnel events firing, ph param now riding on phone-submitting Lead events)
- Shared FUB helper: `src/lib/fub.ts`
- FUB API key already in Vercel env from prior build
