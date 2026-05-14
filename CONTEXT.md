# HGPG Context (Brain)

Last updated: 2026-05-13

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## What's active right now

- **CMA Engine** (cma.homegrownpropertygroup.com) - active build, see `projects/cma-engine.md`
- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, see `projects/transaction-manager.md`
- **TC Concierge** (tc.homegrownpropertygroup.com) - intake, live with Don running real deals
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - blocked on GitHub auth, see `projects/listing-report-portal.md`
- **Claude skills** - five new skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer)

## Parked - run on a date

- **New Construction phone capture rate review** - run between 2026-05-19 and 2026-05-26, see `projects/new-construction-phone-capture-rate-review.md`. Pull FUB data after 7-14 days post-shipping, evaluate if optional-phone nudges are pulling weight, decide if iteration needed.
- **New Construction "For Builders" footer link** - low priority, bundle with next NC site touch, see `projects/new-construction-builder-submit-footer-link.md`. Adds a discreet footer link to /builder-submit so builder reps can find the incentive submission form without Brian manually sharing the URL.

## Recently completed

- **New Construction SMS speed-to-lead** (2026-05-13) - Builder Intro instant SMS via Lead Flow + 5-min backup task via Automation 2.0, see `projects/new-construction-sms-speed-to-lead.md`. **Live end-to-end test pending** (Brian had to jet before final test).
- **New Construction phone capture** (2026-05-12) - tiered downgrade, see `projects/new-construction-phone-capture.md`
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite
- Sellers guide rebrand to brand colors

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- GitHub auth not configured on Mac Mini (blocks Listing Report Portal pushes)
- Sherlock 403 on Transaction Manager (likely API key scope)
- .net Google Workspace migration to .com (do not proactively remind)
- GitHub PAT exposed in chat history needs rotation

## Standing rules learned this week

- **FUB Lead Flow conditions are restricted** to: Tags, Price, City, State, ZIP, MLS, Phone Number. Custom fields are NOT filterable at Lead Flow.
- **FUB Automations 2.0 have NO Send Text step.** Only Lead Flow can send native auto-SMS.
- **FUB custom field API names preserve uppercase runs in labels.** Always GET /customFields to verify after creating.
- **FUB send-from number for new construction:** (980) 261-9222
- **555-pattern phones get flagged invalid in FUB** and won't actually receive SMS. Use real phones for tests.
- **Builder-rep submission page lives at `/builder-submit`** on the new construction site (unlinked from nav by design - shared back-channel for now)

## Where to look next

| Topic | File |
|---|---|
| Brand colors, fonts, voice | `brand.md` |
| Team, FUB IDs, contractors | `team.md` |
| GitHub/Vercel/Supabase/DNS | `infrastructure.md` |
| SC compliance, Lando Law | `compliance.md` |
| Meta Ads, lead re-engagement | `marketing.md` |
| Session protocol, formatting, iMessage | `operations.md` |
| Specific project state | `projects/<name>.md` |
