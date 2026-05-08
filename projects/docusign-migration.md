<!-- Last Updated: 2026-05-08 (originally specced 2026-05-06) -->

# DocuSign Migration off zipForms

- **Status:** 🔵 Workflow scoped, no build, no template recreation yet
- **Goal:** Eliminate zipForms entirely, consolidate all form workflows to DocuSign

## Why this is parked

Currently using zipForms (Lone Wolf Transactions) for templates and DocuSign for signatures — two tools where one would do. DocuSign already has the blank NC and SC association forms accessible. The work is rebuilding the zipForms templates (pre-filled prompts, signing order, role assignments) inside DocuSign.

## Hard truth: no direct export

zipForms does not expose a "send templates to DocuSign" path. The forms are licensed from NC/SC Realtor associations and walled off by design. Recreation by hand is the only option.

But because the blank forms are already accessible in DocuSign, this is rebuilding template behavior, not redrawing forms.

## The path: DocuSign Templates

DocuSign Templates do exactly what zipForms templates do:

1. Open the blank form in DocuSign (e.g. NC Offer to Purchase, SC Agreement to Buy and Sell, NCAR Buyer Agency)
2. Save as Template instead of sending
3. Configure:
   - **Roles:** Buyer 1, Buyer 2, Seller 1, Listing Agent, Buyer Agent (same concept as zipForms parties)
   - **Signing order:** match NC/SC norms
   - **Field tags:** every signature, initial, date, fillable text box
   - **Pre-filled fields:** for anything always Brian (name, license number, brokerage info, Real Broker LLC details, email/phone)
   - **Document name pattern:** e.g. "{{Property Address}} - Offer to Purchase"
4. Save, then duplicate logic across each form

When a deal comes in: "Use Template" → fill property/parties → send. Same flow as zipForms "create from template," fewer clicks.

## Bundling (the zipForms killer feature)

zipForms lets you stack forms into one transaction. DocuSign equivalents:

- **Composite Templates:** multiple template + new doc combinations sent as one envelope
- **Template + Template:** two pre-built templates merged at send time
- Document name normalization: ensures all bundled docs reference the same property/parties

## Build packages (priority order)

| Package | Forms included | Priority |
|---|---|---|
| **NC Listing** | Form 101 (Exclusive Right to Sell), Forms 1-5, Form 10, VLDS, brokerage/HGPG PDFs (4) | 1st |
| **NC Buyer** | NCAR Buyer Agency, Offer to Purchase, addenda, brokerage/HGPG PDFs | 2nd |
| **SC Listing** | SCR forms (verify Realtor Edition includes them) | 3rd |
| **SC Buyer** | SC Agreement to Buy and Sell, addenda, SCR forms | 4th |

## Build checklist (per package)

For each NCAR/SCR form (1-5, 10, VLDS, etc.):
- [ ] Add to template from DocuSign forms library
- [ ] Verify pre-fill defaults for HGPG/agent fields
- [ ] Tag any custom fields not pre-tagged by association
- [ ] Test send, verify all fields render
- [ ] Save to "[Package Name]" template folder

For each brokerage/HGPG PDF:
- [ ] Pull current PDF from source
- [ ] Upload to DocuSign as template
- [ ] Tag every signature, initial, date with role assignment
- [ ] Tag every fillable text field
- [ ] Set pre-fill defaults
- [ ] Test send
- [ ] Save to package folder

For each bundle:
- [ ] Configure multi-template send (or composite template)
- [ ] Verify signing order across all docs
- [ ] Test full bundle send to test email
- [ ] Document send flow in Don's TC playbook

## Pre-build prep

- **Account-level pre-fill defaults:** Look for "Custom Fields" or "Saved Fields" in DocuSign account settings. Set Brian (license, brokerage, contact) once. Auto-populates across every template instead of typing it 11 times.
- **Realtor Edition check:** NCAR forms in Realtor Edition usually come pre-tagged for signatures, initials, common text. Check before manually tagging.
- **SC verification:** Confirm SC Realtor Edition exists / includes SCR forms before scoping SC Listing build.

## Stress test gate

Before cancelling zipForms subscription:

- [ ] One real listing run end-to-end through DocuSign-only flow
- [ ] All 4 packages built and tested
- [ ] One of each package run live (1 NC Listing, 1 NC Buyer, 1 SC Listing, 1 SC Buyer)

Keep zipForms subscription active until all gates pass. Cost of one extra month is far less than the cost of being mid-deal when you discover a missing template.

## Open questions

- Does SC have a Realtor Edition equivalent for SCR forms? Determines whether SC Listing build is the same workflow or involves PDF uploads.
- MLS auto-fill: zipForms pulls property data from MLS automatically. DocuSign does not. Spec future tooling here — possibly a Vercel function that takes an MLS number and returns the property fields, then a Chrome extension or browser bookmarklet to paste into DocuSign. Not v1 scope.

## TC integration (future scope)

- Don's TC Concierge currently extracts intake PDFs from emails. Could those automatically attach to a DocuSign envelope for the matching transaction? Would tighten the workflow, but probably v2.

## Pickup notes

- Start with NC Listing Form 101 — heaviest form, surfaces any tagging/role-assignment quirks before wasting time on lighter forms
- Each package is ~2-4 hours of careful template building. Budget a half-day per package.
- Real-deal stress test before subscription cancellation is non-negotiable
