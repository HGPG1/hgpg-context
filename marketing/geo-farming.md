<!-- Last Updated: 2026-05-17 -->

# Geographic Farming

Direct-mail farming program targeting higher-price-point neighborhoods within ~15 minutes of HGPG HQ. Year 1 launches June 2026 with three farms across Indian Land and Lancaster.

## Active Farms (Year 1)

| Farm | Zip | Median Sold | Annual Sales | Top Competitor | Homes |
|---|---|---|---|---|---|
| Bent Creek | 29720 Lancaster | $785,000 | 19 | Sharon Snipes (EXP Rock Hill, 10%) | ~430 |
| Bridgemill | 29707 Indian Land | $819,990 | 19 | Keith Hensel (Helen Adams, 15%) | ~600 |
| Queensbridge | 29707 Indian Land | $724,500 | 10 | Jessica Mangan (Realty One, 7%) | ~315 |

**Total Year 1 mailer spend:** ~$7,900 (1,345 homes x 12 mailings x ~$0.49/piece including postage)

**Combined addressable commission flowing through these neighborhoods (12-month basis):** ~$1.1M at HGPG's 2.75% per side.

## Watch List

| Farm | Zip | Why Not Yet |
|---|---|---|
| The Retreat at Rayfield | 29707 Indian Land | Kimberly Magette (KW South Park) holds 25% share, currently has 3 of 8 active/pending listings. Launch in Year 2 OR with differentiated content angle (market updates, not just listing announcements). |
| The Estates at Sugar Creek | 29707 Indian Land | 100% Taylor Morrison builder sell-out, Cathy Whiteside has 60 of 67 closings. Median DOM 1 day = pre-sold. Revisit Q1 2028 when 3-year resale wave begins. |

## Supabase Tracking (HGPG Core)

Project: `ioypqogunwsoucgsnmla` (HGPG Core)

Tables:
- `farm_neighborhoods` - one row per farm, holds baseline stats and assignment
- `farm_campaigns` - one row per mailer drop, includes `qr_code_slug` for attribution
- `farm_leads` - one row per inbound response (QR scan or form submission)
- `farm_campaign_roi` (view) - rolls up spend, leads, closings, commission, ROI per farm

Run anytime to check program health:

    SELECT * FROM farm_campaign_roi;

### Seeded neighborhood IDs

    Bent Creek (29720):        4f3b0cbf-8009-4573-a130-c1df11a91a28
    Bridgemill (29707):        d8983d8c-8004-4f41-8cae-367f013340d7
    Queensbridge (29707):      5114a40e-1bc4-4701-9451-64a12b575086
    Rayfield (29707) [watch]:  b41698a9-3be1-42d9-94fd-6a5c4731878b
    Sugar Creek [watch]:       a81c3d16-5fbc-4038-9cc9-87627e8ab067

## Landing Pages (homegrownpropertygroup.com)

Route: `/farm/[slug]` (LIVE as of 2026-05-15, FUB push wired 2026-05-17)

Each mailer's QR code encodes a URL like `https://homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06`. The slug identifies the farm, the `c=` param links to `farm_campaigns.qr_code_slug` for attribution. Scans are logged server-side. Form submissions write to `farm_leads` AND push to FUB.

Code lives in `HGPG1/homegrown-property-group-site` on master:

- `app/farm/[slug]/page.tsx` - dynamic landing page (server component, logs QR scan)
- `app/farm/[slug]/FarmLandingClient.tsx` - form UI in HGPG brand colors
- `app/api/farm/lead/route.ts` - form submission API (Supabase insert + FUB event push)

Implementation note: uses Supabase REST and FUB REST via plain fetch (no `@supabase/supabase-js` dep, since the site doesn't carry the SDK). Required env vars in Vercel: `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` (set 2026-05-15), and `FUB_API_KEY` (verified 2026-05-17).

### FUB integration

Form submissions fire `POST /v1/events` to FUB (not `/v1/people`) so the lead gets proper routing, source attribution, and Brian's new-lead notification. Each submission produces a FUB person with:

- **Tags:** `Geo Farm`, `<Neighborhood>` (e.g. `Bent Creek`), `Touch 1 - Intro`
- **Source:** `Geo Farm - <Neighborhood>` (e.g. `Geo Farm - Bridgemill`)
- **Stage:** `Lead`
- **Assigned to:** Owner Account (Brian) - HQ-led for Year 1
- **Address:** stored as `home` type (FUB has no `property` type)

FUB push is non-blocking: if FUB is down or returns an error, the Supabase `farm_leads` row is still the source of truth and the form still confirms to the user. The route writes `fub_person_id` back to `farm_leads` on success for audit. The route also tolerates FUB's inconsistent event-response shape (some versions return `data.person.id`, others don't) and falls back to an email lookup against `/v1/people` to resolve the FUB id.

For future Touch N campaigns, the tag pattern stays the same (`Touch 2 - Market Update`, etc.) and `mergeTags=true` is implicit (FUB API default), so a returning visitor gets the new tag appended rather than replacing prior tags.

End-to-end verified 2026-05-17 with two real form submissions: Bent Creek + bc-2026-06 attribution → FUB person 32072, Bridgemill + bm-2026-06 attribution → FUB person 32073. Both rows linked correctly in `farm_leads.fub_person_id`. Test data has since been deleted from `farm_leads`; test FUB persons are tagged `DELETE-ME-test-data` for FUB-UI bulk cleanup.

## 12-Month Content Calendar

Same five-theme rotation across all three farms, same design system, three farm-variant designs per send.

| Month | Theme | Hook |
|---|---|---|
| June 2026 | Introduction | Meet the local team |
| July 2026 | Market Update Q2 | What sold this quarter |
| August 2026 | Just Sold | Another happy seller |
| September 2026 | Neighborhood Story | Why this market is strong |
| October 2026 | Value-Add | Fall maintenance checklist |
| November 2026 | Market Update Q3 | Q3 numbers and what they mean |
| December 2026 | Just Sold + Holiday | Year-end thank you |
| January 2027 | Year in Review | Every 2026 sale, every story |
| February 2027 | Spring Prep | Thinking about selling this spring? |
| March 2027 | Just Listed / Spring | Coming soon OR spring market |
| April 2027 | Market Update Q1 2027 | Q1 closings + spring inventory |
| May 2027 | Anniversary | One year of results, invitation to coffee |

## Mailing Lists

Starting universe pulled from MLS Grid (HGPG Listing Reports + MLS project, `mls_property` table). These represent **homes that have ever been listed** - they capture most of each farm but miss original-owner properties that never sold.

Files in iCloud workspace at `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/`:

- `farm-addresses-bent-creek.csv` (227 addresses)
- `farm-addresses-bridgemill.csv` (524 addresses)
- `farm-addresses-queensbridge.csv` (232 addresses)

**To get a complete mailing universe**, either:

1. Layer in Lancaster County tax-roll data (county GIS exports parcel + owner)
2. Use EDDM (Every Door Direct Mail) saturation through any vendor - bypasses the address-list problem entirely

## Print Vendor (LOCKED 2026-05-15, switched to LITE)

**Geosential LITE** at $47/month ($564/year). Postcard-only tier of Janine Sasso's Geosential platform, which is a white-labeled Mailbox Power reseller.

### Why this vendor

After pricing all named alternatives at our actual volume (1,345 homes × 12 sends = 16,140 pieces/yr), LITE is the cheapest option AND covers the features we actually need. Full numbers:

| Plan | Platform/yr | Postage/yr | **Total Year 1** |
|---|---|---|---|
| **Geosential LITE** ✅ | $564 | $12,589 | **$13,153** |
| Mailbox Power Pro annual | $990 | $12,589 | $13,579 |
| Geosential PRO annual | $1,988 | $12,589 | $14,577 |
| ProspectsPLUS! standard | included | included | $12,589 |
| ProspectsPLUS! first class | included | included | $15,010 |
| Wise Pelican 6x9 jumbo | included | included | $16,786 |
| Geosential ELITE (one-time setup) | $6,000 + $1,988 yr 2+ | $12,589 | $18,589 yr 1 |

### What LITE includes

- Unlimited free 4x6 and 5.5x8.5 postcards (we pay only USPS postage)
- Postage: $0.78/piece on 5.5x8.5 jumbo, $0.61/piece on 4x6
- Smart QR codes with real-time text scan notifications to Brian
- USPS Informed Delivery integration (need to verify exact behavior - cosmetic optimization vs full ride-along color preview)
- List builder access (~$0.10/contact)
- Automation builder for scheduling campaigns
- 6 stock postcard templates (we will use our own Canva designs instead)
- Weekday chat support (Mon-Fri)
- Month-to-month, cancel anytime

### What LITE excludes (and why that is fine for HGPG)

- Janine's private templates → we are designing in Canva on HGPG brand
- Full CRM and dashboard → we have FUB
- 15+ lead generation funnels → we have /farm/[slug] landing pages
- Website builder + Google reviews management → we have homegrownpropertygroup.com
- Live chat + popup widgets → not part of our funnel
- Integrated inbox (email/SMS/FB/Google) → FUB handles this
- Client gifting + birthday automations → we can add this separately if it becomes valuable
- Done-for-you setup, 1:1 calls with Janine → we have technical infrastructure already built
- 7-day direct community access to Janine → not a fit for our workflow

### Pre-signup verification (CONFIRMED 2026-05-16 by Janine Sasso directly)

Five questions answered:

1. **USPS Informed Delivery on LITE: grayscale only.** No color ride-along. Skip this as a selling point - it does not provide a meaningful second digital touch on LITE.
2. **Google Street View auto-merge on LITE: yes.** Each postcard shows recipient's actual home. This is the real differentiator and is included.
3. **List builder pricing on LITE: $0.10/contact.** Market rate, confirmed.
4. **Custom design upload on LITE: yes.** We can upload our own Canva-designed PDFs in addition to the 6 stock templates. Not locked in.
5. **Scan data export on LITE: no.** CSV/webhook export is gated to PRO. We DO NOT rely on Geosential for scan tracking - see "Scan tracking architecture" below.

Janine's framing: *"LITE is a tool, PRO is a system."* That is correct, and is exactly why LITE is right for HGPG - we have already built the system (FUB, /farm landing pages, Supabase farm_leads, brain repo, etc.). We need a tool that prints and mails. LITE is that tool.

### Scan tracking architecture (why LITE is enough)

Our QR codes encode URLs in our own domain: `https://www.homegrownpropertygroup.com/farm/<slug>?c=<campaign-slug>`. Every scan hits our Next.js route, our server logs it to `farm_leads`, and the `farm_campaign_roi` view rolls it up. Geosential never sees this data - they only know "a card was mailed." We are the canonical source of scan + attribution data.

Two consequences:

- **We lose Geosential's text-on-scan notification feature** (PRO-only via their system anyway, and would alert based on their tracking, which we are not using). If real-time scan alerts become valuable, we rebuild it as a Supabase trigger → Resend/Twilio → Brian's phone (~30 min of work). Add to deferred items if/when it matters.
- **Geosential's dashboard becomes reference-only.** Use it for design management, list management, print scheduling. Not for analytics. Our Supabase view is canonical.

This is the right architecture: Geosential is interchangeable. If we ever switch vendors (Mailbox Power direct, ProspectsPLUS!, etc.), our scan/lead data and attribution survive untouched because they live in our own infrastructure.

### Vendors not chosen and why

- **Geosential PRO** - $1,424/yr more than LITE. Worth it for an agent without infrastructure (CRM, website, funnels). Redundant for HGPG which already has FUB, homegrownpropertygroup.com, /farm landing pages, and Supabase farm_leads.
- **Geosential ELITE** - $6,000 one-time install. For agents who want Janine to personally build their farming system over 12 1:1 calls. We have the technical chops to skip this.
- **Mailbox Power Pro direct** - $990/yr (vs $564 for LITE). Adds 50 free greeting cards/month for client gifting. Worth considering if HGPG starts a separate sphere-of-influence card program; for farming-only it is unnecessary.
- **Wise Pelican** - $3,633/yr more, no Street View merge, no platform-level automation. Could be a fallback if Geosential becomes problematic.
- **ProspectsPLUS!** - Comparable cost, no Street View, dated templates, less differentiated.
- **Corefact / Cactus Mailing** - Higher cost or less real-estate-specific.

### Janine Sasso context

Janine Sasso runs The Hyper Local Agent training brand and built Geosential as a Mailbox Power-powered platform (Mailbox Power runs a public white-label agency program). She appears as "Janine S. Texas" in Mailbox Power's marketing testimonials. Her business model bundles platform access + training + community + done-for-you services into three tiers (LITE/PRO/ELITE). For our use case, LITE captures the underlying tool without paying the bundle premium for services we do not need.


## Key Principles

1. **Consistency beats volume.** 12 mailings/year to 1,345 homes outperforms 100,000 one-time. Most agents quit at month 4 - that's the moat.
2. **Lead with data, not selling.** Mailer-fatigued homeowners ignore "I'll sell your home!" Real numbers earn attention.
3. **One design system across all farms.** Three variants, one print order, three address files. Geographic and brand synergy.
4. **Always include a tracked QR.** Attribution is the difference between a science and a hope.
5. **Real Broker compliance.** Every piece includes broker name and address.

## Output Files (PDFs in iCloud workspace)

- `HGPG_Farm_Analysis_2026-05-15.pdf` - original four-neighborhood analysis
- `HGPG_Farm_Plan_2026-05-15.pdf` - final three-farm plan with calendar and Supabase tracker reference

## Next Decisions

- [x] Pick vendor - LOCKED 2026-05-15: Geosential LITE ($47/mo, $564/yr)
- [x] Build FUB integration in `/api/farm/lead/route.ts` - shipped 2026-05-17, end-to-end verified
- [x] Decide on agent assignment per farm - HQ-led for Year 1 (logged 2026-05-17). All three farms route to Owner Account (Brian) via FUB default assignment. Revisit Q3 2026 once lead volume per farm is known. To switch later: update `farm_neighborhoods.assigned_agent_id` AND change the route or rely on FUB lead-flow rules.
- [x] June 2026 campaigns logged in `farm_campaigns` (2026-05-17): `bc-2026-06`, `bm-2026-06`, `qb-2026-06`. `cost_total` placeholder = 0; update with `UPDATE farm_campaigns SET cost_total = X WHERE qr_code_slug = ...` once Geosential invoices land.
- [ ] Decide whether to supplement MLS-derived address lists with county tax roll (or use EDDM saturation)
- [ ] Order June 2026 mailers from Geosential (campaigns already logged in DB; just need the physical drop)
- [ ] **July or later: set up USPS Informed Delivery color ride-along** via https://www.usps.com/business/informed-delivery.htm (Mailer Campaign Portal). Free direct from USPS, gives recipients a branded color preview image + clickable URL in their daily Informed Delivery email instead of the default fuzzy grayscale scan. Effectively a free second digital touch per drop. Not for June launch (too much new at once), defer until design system is locked and we have rhythm.

