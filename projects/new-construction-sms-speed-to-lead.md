<!-- Last Updated: 2026-05-12 -->

# New Construction Site - SMS Speed-to-Lead (Builder Intro)

**Status:** 📋 SPEC LOCKED, STAGE 1 READY TO SHIP
**Repo:** `HGPG1/charlotte-new-construction-nextjs` (branch: `main`)
**Domain:** `newconstruction.homegrownpropertygroup.com`
**Driver:** Builder Intro is the hottest funnel surface (hand-raise). Get an instant SMS auto-response on submission so HGPG is first in the door before the builder rep calls.

## Architecture decision (2026-05-12)

**Path A — FUB Lead Flow + Action Plan**, chosen over Path C (LoopMessage from API route).

**Why:**
- FUB-native = no new code on the messaging path, message copy editable from FUB UI without a deploy
- All leads get the SMS (iMessage + Android, unlike LoopMessage which is iMessage-only)
- Built-in STOP/opt-out handling
- Consistent with how the incentives forms already use FUB Automations 2.0

**TCPA gate:** SMS only fires when `customSmsConsent` field on the FUB contact equals `YES`. This is the defensible audit trail.

## Critical pre-work finding

`src/lib/fub.ts` currently writes SMS consent into the FUB Event **message body** as a freetext `SMS Consent: YES/NO` line. FUB Lead Flow and Automations cannot filter on text inside the message body. So **Stage 1 (code) must ship before Stage 2 (FUB UI) can be built.**

## Build sequence — two sequential stages

### Stage 1 (code PR) — Write SMS consent to FUB custom field

Change in `src/lib/fub.ts` `postLeadToFub`: add `customSmsConsent: smsConsent === true ? 'YES' : 'NO'` on the `person` payload. FUB auto-creates custom fields on first Events API write - no manual provisioning. Keep the existing freetext line in the message body for human readability.

Write `'NO'` explicitly on email-only / consent-not-given leads (not undefined / null) so Lead Flow conditions are binary instead of ternary.

**While in there:** read the current SMS consent checkbox label copy in `BuilderLeadModal.tsx` and `LeadCaptureModal.tsx`. If it doesn't explicitly say "Home Grown Property Group", describe what messages will be sent, mention msg/data rates, and include opt-out language, fix it in the same PR. TCPA requires clear consent language.

### Stage 2 (FUB UI) — Lead Flow rule + Action Plan

Cannot start until Stage 1 has shipped and at least one real lead has populated the `SMS Consent` custom field. FUB won't surface custom fields as filter options in Lead Flow conditions until at least one record has the field set.

**2a.** Verify the Builder Intro event arriving in FUB - confirm exact source string, event type (must be one of `Registration`, `Property Inquiry`, `Seller Inquiry`, `General Inquiry` to be Lead Flow eligible), and tag combo.

**2b.** Action Plan `New Construction - Builder Intro - Instant SMS`:
- Step 1: Send Text Message, Delay: Immediately
- Message: `Hi {{firstName}}, Brian from Home Grown Property Group. Thanks for the builder intro request. I'll reach out shortly with next steps. Reply STOP to opt out.` (155 chars, single segment, brand-voice, no em dashes, sender ID + opt-out included)
- Step 2: Create Task, Delay: 5 min, assigned to Brian as backup

**2c.** Lead Flow rule `New Construction - Builder Intro - Consented`:
- Conditions (ALL): source = New Construction Site, event type = (verified), tag contains `Builder Intro`, custom field `SMS Consent` = `YES`
- Action: Apply Action Plan + Assign to Brian
- Position: ABOVE any existing catch-all NC Lead Flow rule

**2d.** Defensive parallel rule `New Construction - Builder Intro - No Consent`:
- Same conditions but `SMS Consent` = `NO`
- Action: Apply Action Plan that does NOT include SMS (email or task only)
- Guardrail against future form changes accidentally creating TCPA exposure

## Why we cannot skip the consent gate

Earlier the question was "gate on consent vs fire on Builder Intro submission regardless?" The consent gate is functionally identical TODAY (Builder Intro requires phone + consent, so the same set fires either way), but the gate protects every FUTURE lead form. The moment a 5th form ships, or Builder Intro becomes phone-optional during a test, an ungated automation creates TCPA exposure on day one. Gate-on-consent makes the system safer-by-default forever. Free defensibility.

## TCPA compliance checklist

| Requirement | How met |
|---|---|
| Prior express written consent | SMS consent checkbox + `customSmsConsent: YES` field |
| Clear consent language on form | TODO: verify current checkbox copy in Stage 1 PR |
| Timestamp + IP captured | FUB native (contact created with sourceUrl + timestamp) |
| Sender ID in message | "Brian from Home Grown Property Group" in copy |
| Opt-out language | "Reply STOP to opt out" in copy |
| STOP honored within 10 biz days | FUB native (auto-blocks outbound to STOP-replied contacts) |
| Audit trail | `customSmsConsent` field + freetext message body line = two artifacts |

## Test plan

**Stage 1 verification:**
1. Submit Builder Intro with phone + consent -> FUB contact `SMS Consent` = `YES`
2. Submit Guide Delivery, no phone -> `SMS Consent` = `NO`
3. Submit Guide Delivery, phone but consent unchecked -> `SMS Consent` = `NO`

**Stage 2 verification:**
1. Submit Builder Intro on live site with own phone -> SMS received within 60 sec
2. Open contact in FUB -> Action Plan visible in timeline
3. Reply STOP -> FUB marks contact as SMS-opted-out
4. Wait 5 min -> backup task appears assigned to Brian

**Edge case:** Manually flip a test contact's `SMS Consent` to `NO`, re-trigger Builder Intro event -> confirm No-Consent rule fires instead of SMS rule.

## What changes vs today

| | Today | After both stages |
|---|---|---|
| Consent captured | Text in event body | Text + queryable custom field |
| FUB can query consent | No | Yes |
| Speed-to-lead SMS | None | Within seconds |
| Backup task | Manual | Auto at 5 min |
| TCPA defensibility | Good | Excellent (two artifacts + gated automation) |
| Channel coverage | N/A | iMessage + Android both |

## References

- Phone capture project: `projects/new-construction-phone-capture.md` (parent build, shipped 2026-05-12)
- FUB Automations 2.0 already in use for incentives forms (variant-tagged Phase 1 drips)
- LoopMessage approach (Path C) rejected: iMessage-only, less defensible audit pattern, more code surface
- FUB docs: action plan text messages don't fire when triggered by Automations - must use Lead Flow + Action Plan path for native SMS auto-send
