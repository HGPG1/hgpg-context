# Incentives Funnel

**Last updated:** 2026-05-05
**Status:** Phase 1 live, monitoring for 2-4 weeks before designing Phase 2

---

## Overview

Paid-traffic-optimized lead capture funnel from Meta ads to the New Construction site. Top-of-funnel page is `newconstruction.homegrownpropertygroup.com/incentives`. Captures first name + email (no phone) and routes to FUB with variant-specific tags so drip campaigns can match the message of the ad the lead clicked.

This is the live ad funnel. It's separate from the older site-wide automations (Guide Delivery / Calculator / Quiz / Builder Intro) which fire from the other surfaces of the New Construction site.

---

## Variant strategy

Three Meta ad creative angles, identified by `utm_content` on the ad URL. Each maps to a distinct FUB tag, which drives a variant-specific Phase 1 drip.

| utm_content value | FUB tag | Creative angle |
|---|---|---|
| `varianta_dollar` | `Variant-A-Dollar` | Dollar-savings framing ("save up to $50,000") |
| `variantb_rate` | `Variant-B-Rate` | Rate-buydown framing ("4.875% builder-paid rate") |
| `variantc_choice` | `Variant-C-Choice` | Choice frame, covers both audiences. Also fallback for missing/null/unrecognized values. |

The mapping logic lives in `src/lib/fubLight.ts` (`variantTagFromUtmContent`). Variant-C is the safe fallback because it works whether the lead is dollar- or rate-motivated.

---

## Form variants on the page

Three form placements all post to `/api/incentives-lead`. The `form_id` field distinguishes them:

| form_id | Where it lives | FUB tag |
|---|---|---|
| `hero` | Top-of-page email capture | `Incentives Form: hero` |
| `card_modal` | Opens when user clicks "Get details" on a specific offer card. Also captures `interested_offer_id`, `interested_builder`, `interested_offer_value`. | `Incentives Form: card_modal`, `Card-Modal-Submit`, `Interested: <Builder>` |
| `secondary` | Bottom-of-page email capture | `Incentives Form: secondary` |

For card_modal submits, the offer interest is also written to the FUB custom field `customOfferInterest` ("Builder ($Amount)" format) so Phase 1 emails can merge it into copy.

---

## Phase 1 automations (live in FUB Automations 2.0)

Created 2026-05-04. Four automations live:

1. **New Construction Incentives — Phase 1 — Card Modal** (triggered by `Card-Modal-Submit` tag — this lead saw a specific offer)
2. **New Construction Incentives — Phase 1 — Variant A Dollar** (triggered by `Variant-A-Dollar` tag)
3. **New Construction Incentives — Phase 1 — Variant B Rate** (triggered by `Variant-B-Rate` tag)
4. **New Construction Incentives — Phase 1 — Variant C Choice** (triggered by `Variant-C-Choice` tag)

Each is a 3-step sequence. Engagement metrics in the FUB UI as of launch are too small to read (single-digit lead counts on automations less than 24 hours old).

---

## NeverBounce email validation (added 2026-05-05)

Real-time email validation on all 3 form variants before lead lands in FUB.

- Validates on blur via server-side `/api/validate-email` (NeverBounce v4)
- Hard-blocks submission for `invalid` and `disposable` results
- Yellow warning + allow for `catchall`, `unknown`, and `spelling_mistake` typos
- FUB receives raw NeverBounce result code in `customEmailValidationStatus` (text field)
- Auto-tags `Email-Invalid` or `Email-Disposable` for downstream filtering

UI permissive on typos (lets real humans through), analytics honest in FUB (raw `invalid` code stored regardless of UI softening). See SESSION-HANDOFF for full implementation notes.

---

## Phase 2 (not designed yet — by design)

There is no documented Phase 2 spec. The variant tag scaffolding was always intended to enable Phase 2 design *after* Phase 1 produces real data.

**Decision logged 2026-05-05:** monitor Phase 1 for 2-4 weeks, then design Phase 2 based on what the data shows. Calendar checkpoint: ~2026-06-02.

### Possible Phase 2 directions (record only, not committed)

- **Re-engagement drips** for leads who went cold after Phase 1 — variant-specific copy that matches the original ad angle
- **Cross-variant moves** if Variant A leads behave more like Variant B over time (e.g. ignore dollar emails, open rate emails)
- **SMS layer** once A2P 10DLC clears (currently blocked on SMS consent checkbox in IDX Broker forms)
- **Stage promotion** triggers (e.g. opened 3+ Phase 1 emails → move to "Engaged" stage → switch to bottom-of-funnel content)
- **NeverBounce-driven branching** — once 50+ leads, decide whether to suppress `Email-Invalid` from drips entirely or keep them in for typo-recovery agent tasks

### Questions to answer before designing Phase 2

- What's the open / click rate per variant? Does any one creative angle clearly win?
- Are leads opening but not converting? (Need lower-funnel content.) Or are they ignoring? (Need different ad creative.)
- What % of leads are flagged `Email-Invalid` / `Email-Disposable`? Above 10% means lead quality issue worth filtering on.
- Are card_modal leads converting at higher rates than hero/secondary? (Tells us whether intent-specific content beats top-of-funnel content.)

---

## Related files

- Page: `src/app/incentives/IncentivesClient.tsx` (in `HGPG1/charlotte-new-construction-nextjs`)
- API route: `src/app/api/incentives-lead/route.ts`
- FUB lib: `src/lib/fubLight.ts`
- Validation route: `src/app/api/validate-email/route.ts`
- FUB email templates: 1147 (Guide Delivery), 1148 (Day 1), 1149 (Calculator), 1150 (Quiz), 1151 (Builder Intro). Note: these are the OLDER NC site templates, not the variant Phase 1 templates — those were created in the FUB UI directly during 5/4 build.

---

## Cross-references

- **HGPG Ads project** owns ad creative production and UTM scheme. Variant naming convention originated there.
- **HGPG Tech & Builds project** owns the page code, API routes, and FUB integration code.
- **HGPG CRM & Leads project** owns the FUB Automations 2.0 builds and email template copy.
- **SESSION-HANDOFF.md** has the most recent build state and the calendar checkpoint for Phase 2 review.
