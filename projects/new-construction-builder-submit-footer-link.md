<!-- Last Updated: 2026-05-13 -->

# New Construction - "For Builders" Footer Link (Parked Follow-up)

**Status:** 📋 PARKED - low priority, do when next touching the new construction site
**Driver:** `/builder-submit` page is live and working but unlinked from the site nav. Currently a back-channel URL Brian shares manually. Adding a discreet footer link makes it discoverable by builder reps without cluttering the buyer-facing nav.

## Current state

- Page lives at `newconstruction.homegrownpropertygroup.com/builder-submit`
- Status 200, title `Submit Your Builder Incentives | HGPG New Construction`
- Form fields: company name, rep name, email, optional phone, incentive type, value, description, expiration date
- Submissions feed `/admin` submissions list (with click-to-call phone numbers per past session)
- NOT linked from main nav or footer anywhere on the site
- Brian shares the URL manually when builder reps ask how to update their incentives

## What to build

Add a "For Builders" link in the site footer of `HGPG1/charlotte-new-construction-nextjs`.

### Placement
Footer (NOT main nav). Buyer-facing nav stays clean - builders aren't a buyer UX consideration.

### Copy options
- "For Builders" (cleanest, single column header style)
- "Builder Reps - Submit Incentives" (more explicit, slightly clunky)
- "Are you a builder?" (question style, conversational)

**Recommended:** "For Builders" as a small footer-column header with the link "Submit Your Incentives" underneath. Discreet, professional, easy to skip past as a buyer.

### Implementation notes
- The footer component lives somewhere in `src/components/` - likely `Footer.tsx` or similar. Grep for the existing footer copy ("Home Grown Property Group" or the address block) to find it.
- Single-line addition, no new component needed.
- Match existing footer link styling (same color, same hover state).
- No analytics event needed (this is a low-traffic link).
- No FUB lead event firing - this is admin form, not a lead capture surface.

## Claude Code prompt (when ready to ship)

```
In HGPG1/charlotte-new-construction-nextjs, find the site footer component (likely src/components/Footer.tsx or in the layout file). Add a discreet "For Builders" section/link to the footer that points to /builder-submit.

Recommended copy:
- Column header: "For Builders"
- Link text: "Submit Your Incentives"
- Link href: "/builder-submit"

Match existing footer link styling. No need for analytics or any new component - this is a single-line addition. Place it as a separate footer column or as a third/fourth link in an existing column, whichever fits the current footer layout cleanly.

Commit message:
    feat: add 'For Builders' footer link to /builder-submit

    The builder incentive submission page is live but was unlinked from the
    site. Adds a discreet footer entry so builder reps can find it without
    cluttering the main buyer-facing nav.

Branch: feature/footer-builder-link
Open a PR against main. Do not merge. Brian reviews.
```

## Why parked, not shipped today

Low impact, low urgency. The page is live and Brian has the URL. Iterating the footer is a 10-minute task best bundled with the next time the new construction site needs a touch. No deploy on its own.

## References

- New construction site: `newconstruction.homegrownpropertygroup.com`
- Repo: `HGPG1/charlotte-new-construction-nextjs`
- Builder submit page (live): `newconstruction.homegrownpropertygroup.com/builder-submit`
- Admin submissions view: `newconstruction.homegrownpropertygroup.com/admin`
- Incentive Concierge pipeline: 3 builders auto-crawl, 7 via email/manual
