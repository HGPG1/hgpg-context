# HGPG - Tech & Builds Handoff: Geo-Farming Infrastructure

## Scope summary

Geographic farming program infrastructure is **mostly complete** as of 2026-05-15. Landing pages are live, attribution tracking works end-to-end, brain doc is committed. This handoff captures (a) what's done, (b) what's pending and worth handling before the June 1 launch, and (c) what's deferred to post-launch iteration.

## What is done and live in production

### Landing pages

Route: `/farm/[slug]` deployed on master in `HGPG1/homegrown-property-group-site`. Verified live 2026-05-15.

URLs:
- https://www.homegrownpropertygroup.com/farm/bent-creek
- https://www.homegrownpropertygroup.com/farm/bridgemill
- https://www.homegrownpropertygroup.com/farm/queensbridge

QR encoding pattern with attribution: `?c=<campaign-qr-slug>` matching `farm_campaigns.qr_code_slug` in Supabase.

Files:
- `app/farm/[slug]/page.tsx` (server component, logs QR scan)
- `app/farm/[slug]/FarmLandingClient.tsx` (form UI in HGPG brand)
- `app/api/farm/lead/route.ts` (form submission API)

Implementation note: uses Supabase REST via plain fetch, NOT `@supabase/supabase-js` (site doesn't carry the SDK). If you need to add Supabase functionality elsewhere on this site, either match this pattern or add the SDK as a dependency (first deploy of the farm code failed because of this).

### Supabase tracking (HGPG Core project `ioypqogunwsoucgsnmla`)

Tables created and seeded:
- `farm_neighborhoods` (one row per farm, baseline stats, agent assignment)
- `farm_campaigns` (one row per mailer drop, includes qr_code_slug for attribution)
- `farm_leads` (one row per inbound response, foreign keys to neighborhood + attributed campaign)

View: `farm_campaign_roi` - rolls up spend, lead counts, closings, commission, ROI per farm.

RLS enabled. Service role key used by Next.js routes. Public anon key has no read access to these tables (lead data is private).

### Env vars (set 2026-05-15)

In Vercel project `prj_aBejJuBUr5BN0atTOiO5xyHL7qxj` on team `team_FietQPKCmnyioG2n0FdteQCV`:
- `NEXT_PUBLIC_SUPABASE_URL` = `https://ioypqogunwsoucgsnmla.supabase.co`
- `SUPABASE_SERVICE_ROLE_KEY` = (from Supabase HGPG Core API settings)

Applied to: Production + Preview + Development.

### Brain repo security patch (`HGPG1/brain-app`)

Added `middleware.ts` at repo root that gates `/api/files` behind Supabase session cookie OR Bearer token. Previously the endpoint was anonymous-readable and leaked the entire private brain repo file tree and contents.

Verified post-deploy: anonymous reads 401, Bearer reads 200, Supabase-cookie-bearing requests 200, write APIs unaffected.

## What is pending before June 1 launch (small, focused)

### 1. Run Supabase INSERT to log June campaigns

Required before drop so attribution works. Run in HGPG Core SQL editor:

    INSERT INTO farm_campaigns (neighborhood_id, qr_code_slug, send_date, theme, format, vendor, quantity, design_notes)
    VALUES
      ('4f3b0cbf-8009-4573-a130-c1df11a91a28', 'bc-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 430, 'Year 1 Touch 1'),
      ('d8983d8c-8004-4f41-8cae-367f013340d7', 'bm-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 600, 'Year 1 Touch 1'),
      ('5114a40e-1bc4-4701-9451-64a12b575086', 'qb-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 315, 'Year 1 Touch 1');

Verify after insert:

    SELECT n.name, c.qr_code_slug, c.send_date, c.quantity
    FROM farm_campaigns c
    JOIN farm_neighborhoods n ON n.id = c.neighborhood_id
    WHERE c.send_date = '2026-06-01';

### 2. FUB push from form submissions (NEW BUILD)

Currently `/api/farm/lead/route.ts` writes to Supabase only. For June launch we want form submissions to also create a FUB person with proper tags so leads land in Brian's pipeline immediately.

Spec:
- On successful form submit, call FUB API (or our existing fub_agent infrastructure) to create a person
- Required fields: first_name, last_name, email, phone, property_address
- Tags to apply: `Geo-Farm`, `<Neighborhood>` (e.g. `Bent-Creek`, `Bridgemill`, `Queensbridge`), `Touch-1-Intro` (or similar campaign identifier)
- Source field: `Direct Mail - Geo Farm <Neighborhood>`
- Failure handling: if FUB push fails, still return success to the user, log the error, alert Brian via email or Slack

Reference: existing FUB integration patterns in `HGPG1/fub_agent` (or whatever the FUB API client lives in). Brain doc `infrastructure.md` should index this.

Acceptance criteria:
- Form submission on a /farm landing page creates a FUB person within 5 seconds
- Person has the correct tags applied
- Source field correctly identifies the geo-farm origin
- A Supabase row in `farm_leads` is still created regardless of FUB success/failure

### 3. Decision: agent assignment per farm

Currently `farm_neighborhoods.assigned_agent_id` is null (all leads route to HQ / Brian). Options:
- Keep HQ-led (Brian fields all leads, distributes manually) - simplest, recommended for Year 1
- Round-robin among Ashley, Brenda, Taylor based on availability
- Geographic split (one agent per farm)

If we switch off HQ-led, update `farm_neighborhoods.assigned_agent_id` and modify the lead route to honor it (notify the assigned agent, not Brian, on new lead).

For June 1 launch, recommend keeping HQ-led. Revisit Q3 2026.

## Deferred enhancements (post-launch, not blocking)

### USPS Informed Delivery ride-along

USPS lets mailers upload a color image + clickable URL that rides along with the daily Informed Delivery email USPS sends to every signed-up household. Free via the Mailer Campaign Portal at https://www.usps.com/business/informed-delivery.htm.

Geosential LITE includes only the default grayscale (which is just baseline USPS behavior, not a Geosential feature). PRO probably handles the ride-along.

To DIY:
1. Register HGPG as a mailer at USPS Mailer Campaign Portal
2. Per campaign: upload a color preview image + URL
3. Link to the campaign's Intelligent Mail barcode

Worth setting up in July or later (after June launch is stable). Effectively a free second digital touch per drop.

### Scan notification system

Geosential LITE does NOT send text alerts when a QR code is scanned (PRO only, but we wouldn't use it anyway since attribution lives in our own Supabase). If real-time scan alerts become valuable, build it ourselves:

- Add a Supabase trigger on `farm_leads` INSERT WHERE `lead_source = 'qr_scan'`
- Trigger calls an edge function that sends a text via Resend/Twilio to Brian's phone (803-902-3700)
- Message: "QR scan: [Neighborhood] - [time]"

Estimated build: 30-60 min. Defer unless Brian explicitly wants scan-time notifications.

### Address universe expansion

MLS-derived address lists cover homes that have ever been listed - roughly 65-75% of any given farm. To reach the full universe:

Option A: Layer in Lancaster County tax-roll data
- Lancaster County GIS exports parcel + owner data
- Match against existing CSVs to find homes we are missing
- Append to mailing list CSV before upload to Geosential

Option B: EDDM (Every Door Direct Mail)
- Submit a saturation list to Geosential / USPS for the carrier routes covering each farm
- Captures every household but loses precision (you mail to renters and apartments too)
- Geosential explicitly does not do EDDM ("not the right way to farm under 1,000 homes")

Recommend Option A for completeness, defer to Q3 2026. Address count delta is small enough that 6 months of mailers to the MLS-derived list is fine.

## Tech reference quick card

- Brain repo: `HGPG1/hgpg-context` (private, editable via brain.homegrownpropertygroup.com)
- Site repo: `HGPG1/homegrown-property-group-site` on master, deploys to Vercel auto
- Site Vercel project: `prj_aBejJuBUr5BN0atTOiO5xyHL7qxj`
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- HGPG Core Supabase: `ioypqogunwsoucgsnmla`
- Brain write API: `https://brain.homegrownpropertygroup.com/api/external/write` (brain repo) and `/api/external/commit` (any HGPG1 repo). Bearer auth via BRAIN_WRITE_TOKEN.

## Acceptance criteria for "launch ready"

- [ ] FUB push wired into `/api/farm/lead/route.ts` and tested with a real submission
- [ ] June campaigns inserted into `farm_campaigns` table
- [ ] Test scan from a real phone on each `/farm/<slug>?c=<slug>` URL writes correct rows to `farm_leads` with attributed campaign linked
- [ ] Agent assignment decision logged in brain (HQ-led for June OR specific agent IDs set on `farm_neighborhoods`)
- [ ] Test form submission on each landing page creates a FUB person with correct tags

## Open questions for Brian

1. FUB API key / credentials path - where does the route file access them? New env var, existing fub_agent SDK, or direct REST?
2. Do we want a Slack alert on new geo-farm lead in addition to FUB, or is FUB notification sufficient?
3. Confirm: HQ-led for Year 1 OR assign per farm?
