<!-- Last Updated: 2026-05-15 -->

# Session Handoff - Geo-Farming Launch

## TL;DR

HGPG is launching a three-farm geographic mailer program June 1, 2026. Infrastructure is built end-to-end (landing pages live, tracking working, brain doc current). Brian needs to verify five questions with Geosential support, sign up for LITE, design postcards in Canva, and ship.

## What is fully done

**Three farms picked, seeded in Supabase HGPG Core (project `ioypqogunwsoucgsnmla`):**
- Bent Creek (29720 Lancaster) - 430 homes - median $785K - id `4f3b0cbf-8009-4573-a130-c1df11a91a28`
- Bridgemill (29707 Indian Land) - 600 homes - median $819,990 - id `d8983d8c-8004-4f41-8cae-367f013340d7`
- Queensbridge (29707 Indian Land) - 315 homes - median $724,500 - id `5114a40e-1bc4-4701-9451-64a12b575086`

**Watch list (not Year 1, revisit later):**
- The Retreat at Rayfield - Kimberly Magette has 25% share (too entrenched for Year 1)
- The Estates at Sugar Creek - 100% Taylor Morrison builder sellout (revisit Q1 2028)

**Supabase tables and view created in HGPG Core:**
- `farm_neighborhoods`, `farm_campaigns`, `farm_leads`
- `farm_campaign_roi` view rolls up spend, leads, closings, commission, ROI per farm
- Run anytime: `SELECT * FROM farm_campaign_roi;`

**Landing pages LIVE in production at homegrownpropertygroup.com:**
- `/farm/bent-creek`, `/farm/bridgemill`, `/farm/queensbridge`
- QR URL pattern: `https://www.homegrownpropertygroup.com/farm/<slug>?c=<campaign-slug>`
- The `?c=` parameter matches `farm_campaigns.qr_code_slug` for attribution
- Server logs every page hit to `farm_leads` with `lead_source = 'qr_scan'`
- Form submissions write `lead_source = 'landing_page'` rows via `/api/farm/lead`
- Verified end-to-end working as of 2026-05-15

**Code lives in `HGPG1/homegrown-property-group-site` on master:**
- `app/farm/[slug]/page.tsx`
- `app/farm/[slug]/FarmLandingClient.tsx`
- `app/api/farm/lead/route.ts`
- Uses Supabase REST via plain fetch (no `@supabase/supabase-js` dep - site doesn't carry the SDK)
- Vercel env vars set: `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`

**Address CSVs exported and in iCloud HGPG-Cowork workspace:**
- `farm-addresses-bent-creek.csv` (227 unique MLS addresses, expect ~430 homes total in farm)
- `farm-addresses-bridgemill.csv` (524 MLS addresses, ~600 homes total)
- `farm-addresses-queensbridge.csv` (232 MLS addresses, ~315 homes total)
- MLS-derived = homes that have ever been listed. To reach the full universe, layer in Lancaster County tax-roll or use EDDM saturation.

**Brain documentation committed to `HGPG1/hgpg-context`:**
- `marketing/geo-farming.md` - program plan, vendor analysis, locked decisions
- `marketing/farms/june-2026-introduction.md` - June mailer brief (template for monthly briefs going forward)
- `projects/brain-app.md` updated with security note

**Security patch shipped to brain-app:**
- `middleware.ts` deployed to `HGPG1/brain-app` on main
- `/api/files` now requires Supabase session cookie OR Bearer token
- Was previously anonymous-readable (full brain repo leak via GET)
- Verified post-deploy: anonymous reads 401, Bearer reads 200, editor UI still works

## Vendor decision: LOCKED 2026-05-15

**Geosential LITE at $47/month ($564/year)**

This is the postcard-only tier of Janine Sasso's Geosential platform, which is a white-labeled Mailbox Power reseller. We chose LITE because:

- Cheapest option at our volume ($13,153/year all-in vs Mailbox Power Pro at $13,579 and Geosential PRO at $14,577)
- Includes unlimited free 4x6 and 5.5x8.5 postcards (we pay only USPS postage)
- Smart QR with text alerts on scans
- USPS Informed Delivery integration (behavior unverified, see below)
- List builder and automation builder
- Month-to-month, cancel anytime

We skip PRO/ELITE because their platform layer (CRM, funnels, website builder, inbox) is redundant for HGPG - we already have FUB, homegrownpropertygroup.com, /farm landing pages, and Supabase farm_leads.

## What Brian needs to do next

### Step 1: Verify these five questions with Geosential before signing up

Open the chat at https://geosential.com (pink bubble bottom-right) and ask:

1. Does LITE support USPS Informed Delivery ride-along color preview (digital image emailed to recipient morning-of-delivery), or is "Enhanced Impression" just grayscale optimization?
2. Does LITE include the Google Street View auto-merge that shows each recipient's home on the postcard?
3. What is the list-builder pricing on LITE per contact?
4. Can I upload my own postcard designs (PDF/X-1a from Canva) on LITE, or am I locked to the 6 stock templates?
5. Does LITE expose scan data via CSV export or webhook, so I can pipe it into my own CRM?

**If questions 1, 2, 4, and 5 all come back yes → LITE confirmed, proceed to Step 2.**

**If any come back no →** Re-evaluate. Most likely flip is back to Mailbox Power Pro direct at $990/yr if Street View merge or design upload is gated to PRO.

### Step 2: Sign up for Geosential LITE

- URL: https://geosential.com/lite-payment
- $47/month, no annual lock
- Login afterward at https://print.geosential.com/login

### Step 3: Design three June Introduction postcard variants in Canva

- Brand kit: **kAHFKMi4Q7g**
- Size: 5.5 x 8.5 jumbo
- Brief at `marketing/farms/june-2026-introduction.md` in brain repo
- One design system, three variants (one per farm)
- Brand rules: navy #2A384C, steel #A0B2C2, light steel #D1D9DF, off-white #F0F0F0. NO green. No em dashes. Cooper Hewitt (body), Sansita Regular (display).
- Tagline: "Growth Starts Here, At the Roots."
- Footer: Real Broker LLC, 7612 Charlotte Highway, Indian Land, SC 29707
- Export each as PDF/X-1a for print
- Can have Claude review proofs before submitting

### Step 4: Log the three June campaigns to Supabase

Run in HGPG Core SQL editor BEFORE the drop:

    INSERT INTO farm_campaigns (neighborhood_id, qr_code_slug, send_date, theme, format, vendor, quantity, design_notes)
    VALUES
      ('4f3b0cbf-8009-4573-a130-c1df11a91a28', 'bc-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 430, 'Year 1 Touch 1 - introduction with Q1 stats'),
      ('d8983d8c-8004-4f41-8cae-367f013340d7', 'bm-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 600, 'Year 1 Touch 1 - introduction with Q1 stats'),
      ('5114a40e-1bc4-4701-9451-64a12b575086', 'qb-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'geosential-lite', 315, 'Year 1 Touch 1 - introduction with Q1 stats');

QR codes on each variant should encode:

    Bent Creek:   https://www.homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06
    Bridgemill:   https://www.homegrownpropertygroup.com/farm/bridgemill?c=bm-2026-06
    Queensbridge: https://www.homegrownpropertygroup.com/farm/queensbridge?c=qb-2026-06

### Step 5: Upload, proof, schedule, ship

- Upload three CSVs from iCloud HGPG-Cowork workspace to Geosential
- Upload three Canva PDFs as 5.5x8.5 templates
- Enable Smart QR with text alerts routing to Brian (803-902-3700)
- Enable Informed Delivery ride-along if available
- Enable Street View merge on back of card if available (or hold for July if proof looks weird)
- Request proof, review for compliance (broker line, Equal Housing icon, all required elements present)
- Approve, schedule June 1 drop

## Year 1 budget

| Line | Amount |
|---|---|
| Geosential LITE | $564 |
| Postage (16,140 pieces × $0.78) | $12,589 |
| **Total Year 1** | **$13,153** |

Break-even: ~12% of one closing at $780K median × 2.75% commission ($21,450).

## 12-month content calendar

| Month | Theme |
|---|---|
| June 2026 | Introduction |
| July 2026 | Q2 Market Update |
| Aug 2026 | Just Sold |
| Sept 2026 | Neighborhood Story |
| Oct 2026 | Value-Add (fall maintenance) |
| Nov 2026 | Q3 Market Update |
| Dec 2026 | Just Sold + Holiday |
| Jan 2027 | Year in Review |
| Feb 2027 | Spring Prep |
| Mar 2027 | Just Listed / Spring |
| Apr 2027 | Q1 2027 Market Update |
| May 2027 | Anniversary |

Each month gets a brief at `marketing/farms/<month>-<year>-<theme>.md`. Use June as the template.

## Open items deferred (not blocking June launch)

- Add FUB push to `/api/farm/lead/route.ts` so form submissions create a FUB person with proper tags (`Geo-Farm`, `Bent-Creek`, etc.)
- Decide whether to layer Lancaster County tax-roll into mailing universe, or accept the MLS-list ~30% gap and use EDDM for full coverage on certain months
- Decide agent assignment per farm (currently HQ-led). If assigning to Ashley/Brenda/Taylor, update `farm_neighborhoods.assigned_agent_id`
- Re-evaluate vendor at Q1 2027 review

## Key infrastructure references

**Brain repo (canonical source):**
- `HGPG1/hgpg-context` on main
- Editable at brain.homegrownpropertygroup.com
- Local at `~/Documents/hgpg-context`, push with `brain` shell function

**Brain write API (for Claude sessions):**
- `BRAIN_WRITE_TOKEN` stored in Claude memory
- POST `https://brain.homegrownpropertygroup.com/api/external/write` (brain repo only)
- POST `https://brain.homegrownpropertygroup.com/api/external/commit` (any HGPG1 repo)
- Bearer auth required

**Supabase HGPG Core:**
- Project ID: `ioypqogunwsoucgsnmla`
- Tables: `farm_neighborhoods`, `farm_campaigns`, `farm_leads`
- View: `farm_campaign_roi`

**Vercel:**
- Team: `team_FietQPKCmnyioG2n0FdteQCV`
- Site project: `prj_aBejJuBUr5BN0atTOiO5xyHL7qxj`

**Output files in iCloud HGPG-Cowork:**
- `HGPG_Farm_Analysis_2026-05-15.pdf`
- `HGPG_Farm_Plan_2026-05-15.pdf`
- `farm-addresses-*.csv` (three farms)

## What this session got wrong and corrected

1. Initially quoted Wise Pelican at $0.35/piece - actual is $1.04 all-in. Corrected after pricing fetch.
2. Initially recommended Mailbox Power Pro at $990/yr - found Geosential LITE at $47/mo is cheaper for same Mailbox Power underlying tech. Final answer flipped to LITE.
3. Initially suggested staging farms (Bridgemill first, others later) - Brian correctly pushed back that staged starts break the consistency model that makes farming work. All three launch June 1.
4. Claimed Informed Delivery was a clear LITE feature - actually unverified. Listed as one of the five questions to confirm before signup.

## What to ask Brian in the next session

If picking up this conversation cold, the right opening question is:

> "Did Geosential's chat confirm the five LITE features? And did you sign up?"

If yes to both → proceed to designing the Canva proofs.
If no on the Geosential answers → re-evaluate Mailbox Power Pro direct as Plan B.
If signed up but no designs yet → offer to draft them on Brian's behalf.
