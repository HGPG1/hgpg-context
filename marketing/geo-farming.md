<!-- Last Updated: 2026-05-15 -->

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

Route: `/farm/[slug]` (LIVE as of 2026-05-15)

Each mailer's QR code encodes a URL like `https://homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06`. The slug identifies the farm, the `c=` param links to `farm_campaigns.qr_code_slug` for attribution. Scans are logged server-side, form submissions write to `farm_leads`.

Code lives in `HGPG1/homegrown-property-group-site` on master:

- `app/farm/[slug]/page.tsx` - dynamic landing page (server component, logs QR scan)
- `app/farm/[slug]/FarmLandingClient.tsx` - form UI in HGPG brand colors
- `app/api/farm/lead/route.ts` - form submission API

Implementation note: uses Supabase REST via plain fetch (no `@supabase/supabase-js` dep, since the site doesn't carry the SDK). Required env vars in Vercel: `NEXT_PUBLIC_SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` (set 2026-05-15).

End-to-end verified: hitting any farm URL writes a `qr_scan` row to `farm_leads` with the correct neighborhood_id. Form submission tested and writes `landing_page` rows.

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

## Print Vendor Options

### Primary: Mailbox Power (likely via Geosential)

Janine Sasso ("the Mailer Mom") works with **Geosential**, which evidence strongly suggests is a white-labeled **Mailbox Power** reseller. Mailbox Power runs a public white-label agency program; multiple resellers exist under different brands. The underlying platform features match what Geosential markets:

- White-labeled fulfillment (your return address, never the platform's brand)
- Dynamic QR codes with scan-tracking + text notifications
- Auto-merge Google Street View image of each recipient's home
- First-name personalization at scale
- Built-in list builder (homeowners, renters, income, credit score filters)
- ~$0.10 per contact for list rental, postcard pricing varies by volume
- Tight CRM integration patterns (works with FUB)

If interested in Janine's exact stack, sign up directly through Geosential to get her training/templates. If we want better pricing, signing up direct with Mailbox Power skips the reseller margin (typically 15-30%).

### Alternates worth considering

- **Wise Pelican** (~$0.35/piece) - real-estate-specific, no minimum, MLS import, draw-tool for custom areas at $0.10/address, QR codes + landing pages built in. Cleanest design templates in the category.
- **ProspectsPLUS!** - Janine's other commonly cited vendor; deeper real-estate template library, MapMyMail for list-building, strong EDDM support.
- **Corefact** - premium feel, more swag/extras (notepads, calendars, magnets). Higher per-piece cost. Good fit for value-add months (October maintenance checklist, January year-in-review).
- **Cactus Mailing** - generic but reliable; built-in call-tracking phone numbers if we want a non-digital response channel.

### Decision needed

Pick one default vendor and commit to it for 12 months. Switching mid-program costs design-system rebuilds and breaks visual continuity. Recommend Mailbox Power direct (or Geosential if we want Janine's templates) for the dynamic QR + Street View features alone - both make our QR attribution simpler and the mailer harder to ignore.

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

- [ ] Pick vendor (Mailbox Power direct vs Geosential reseller vs Wise Pelican) by ~May 22 to allow June 1 drop
- [ ] Decide whether to supplement MLS-derived address lists with county tax roll (or use EDDM saturation)
- [ ] Build FUB integration in `/api/farm/lead/route.ts` - currently writes to Supabase only
- [ ] Decide on agent assignment per farm (currently HQ-led) - if assigning to Ashley, Brenda, or Taylor, update `farm_neighborhoods.assigned_agent_id`
- [ ] Order June 2026 mailers and log to `farm_campaigns` once they drop
