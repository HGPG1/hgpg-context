<!-- Last Updated: 2026-05-17 -->
# June 2026 Geo-Farming Introduction Postcards

Touch #1 of a 12-month farming program across three Indian Land / Lancaster neighborhoods. Print-ready PDFs built and verified against real MLS data. Awaiting Brian's review + Geosential upload by May 22 for June 1 drop.

## Status

🟡 PDFs built, data verified against Supabase MLS, awaiting Brian's final review.

## Deliverables (session 2026-05-17)

PDFs at `/mnt/user-data/outputs/`:

- `HGPG_postcard_bent_creek_2026-06.pdf`
- `HGPG_postcard_bridgemill_2026-06.pdf`
- `HGPG_postcard_queensbridge_2026-06.pdf`
- `HGPG_postcards_print_spec.md` (Geosential upload checklist)

Source files at `/home/claude/postcards/` (ephemeral session container). Rebuild instructions in pickup notes below.

## Print spec

- Trim: 5.5 x 8.5 in (USPS jumbo postcard)
- Bleed: 0.125 in all sides
- Canvas with bleed: 5.75 x 8.75 in
- Safe area: 0.25 in inside trim
- Resolution: 300 DPI sRGB (Geosential converts to CMYK)
- Stock: standard glossy, full color both sides
- Pages per PDF: 2 (front, back)

## Quantities

| Farm | Print qty | Primary zip |
|---|---|---|
| Bent Creek | 430 | 29720 (some 29707) |
| Bridgemill | 600 | 29707 |
| Queensbridge | 315 | 29707 |
| **Total** | **1,345** | |

## QR routing and attribution

| Farm | URL |
|---|---|
| Bent Creek | `https://www.homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06` |
| Bridgemill | `https://www.homegrownpropertygroup.com/farm/bridgemill?c=bm-2026-06` |
| Queensbridge | `https://www.homegrownpropertygroup.com/farm/queensbridge?c=qb-2026-06` |

All three QR codes scan-verified during build. ECC level H, navy on white (not pure black, per brand), 1.25 in square at 300 DPI.

**Open item for HGPG-Tech:** confirm the `?c=` campaign tag flows through farm landing pages into FUB as `lead_source` or `source_detail` field on form submit. Otherwise we lose per-touch attribution.

## Design system summary

Reusable across all 12 monthly touches.

**Front (top to bottom):**
- Top 55%: photo hero, grayscale treatment, with neighborhood-name typography in white Sansita Bold + "YOUR NEIGHBORHOOD" small-caps accent label above + steel rule + "Monthly Market Brief From HGPG" tracked sublabel
- Middle 18%: navy headline band — "The [Neighborhood] Market, In Plain English." in Sansita Bold white + subhead in Inter steel
- Bottom 27%: continuous navy — combined HGPG + Real Broker LLC lockup centered + steel divider rule + "GROWTH STARTS HERE, AT THE ROOTS." tagline in tracked Inter

**Back (top to bottom):**
- Top 8.5%: navy header band with "[Neighborhood] - Trailing 12-Month Snapshot" in Sansita Bold
- Three light-steel stat tiles: median sold price, sold count, sale-to-list ratio (Sansita Bold values, Inter Bold tracked labels)
- Steel intro panel: 4-line introduction from Brian in 32pt Inter
- Recent sales table: 4 rows, address left + price right + month right, with optional segment label for Bridgemill ("Single Family Residences")
- Navy Quick Take callout: 2-sentence interpretive read on the data in 30pt Inter, with steel left bar accent
- Contact row: round navy-ringed Brian headshot left, name + Team Lead title + phone + email stacked; QR code right with caption above and URL below
- Navy compliance footer: combined HGPG + Real Broker LLC lockup centered, address line below, EHO icon right

## Brand rules honored

- Navy / steel / light steel / off-white palette only — no green, no red
- Sansita Bold for display, Inter for body (fallback for Cooper Hewitt per project standing rule)
- No em dashes anywhere
- "Team Lead, Home Grown Property Group" everywhere (not Broker/Owner)
- Combined HGPG + Real Broker LLC lockup on both sides
- Equal Housing Opportunity icon on compliance footer

## MLS data sources

All stats sourced from `mls_property` table in Supabase project `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS).

Filter logic per farm:
- **Bent Creek:** `LOWER(subdivision_name) ILIKE '%bent creek%' AND postal_code IN ('29720','29707')`
- **Bridgemill:** `LOWER(subdivision_name) ILIKE '%bridgemill%' AND postal_code = '29707'`
- **Queensbridge:** `LOWER(subdivision_name) ILIKE '%queensbridge%' AND postal_code = '29707'`

All queries add: `standard_status = 'Closed'`, `close_price >= 100000` (filters out lease records and data errors), `close_date BETWEEN '2025-05-17' AND '2026-05-17'` for trailing 12-month stats.

### Stats on cards (trailing 12mo as of 2026-05-17)

| Farm | Median Sold | Sold Count | Sale-to-List | Segment |
|---|---|---|---|---|
| Bent Creek | $785,000 | 19 | 98.3% | All SFH |
| Bridgemill | $780,000 | 26 | 98.3% | **Single Family only** (townhomes excluded) |
| Queensbridge | $724,500 | 10 | 98.4% | All SFH |

### Bridgemill segmentation note

Bridgemill is the only farm with both SFH and townhome inventory. SFH and townhome markets do not overlap on price:
- SFH: $584K - $1.175M (median $780K), 26 closings
- Townhome: $390K - $486K (median $438K), 10 closings

Touch #1 reports SFH-only data and labels it as such. Townhome owners receive the card (carrier-route mailing) but see clearly that the data covers SFH. Considered later: dedicated townhome card if scan/conversion data justifies the investment.

## What's still placeholder, may need swap before print

1. **Hero photos** — currently stock-with-grayscale placeholders. Brian to decide whether to source real Bent Creek gate, Bridgemill clubhouse, and Queensbridge home photos. If not swapped, the stock-with-grayscale works as a permanent design system.

2. **EHO icon** — custom-drawn approximation. Acceptable for v1, swap to official Equal Housing Opportunity vector when convenient.

## Monthly production workflow (touches 2-12)

Once data is in hand from Supabase, monthly production is ~20-30 minutes per farm:

1. Re-pull trailing-12-month stats and last 90-day sales using the queries above (advance the date window)
2. Update three stat tiles, four recent sales rows, Quick Take copy
3. Update header band period reference if needed
4. Update campaign tag in QR URLs (`c=<farm>-YYYY-MM`)
5. Re-render PDF, verify QR scans, upload to Geosential

The QR can stay pointed at the same farm-specific lander permanently. Only the `c=` tag rotates.

## Touch arc plan (12 months)

- **Touches 1-4 (Jun-Sep 2026):** "Growth Starts Here, At the Roots." tagline. Build recognition through consistent monthly market data. No claims, no asks.
- **Touches 5-8 (Oct 2026-Jan 2027):** Drop the tagline; let the data carry it. By month 5 the cards are recognized and the brand line is redundant.
- **Touches 9-12 (Feb-May 2027):** Earn the right to switch tone. "Your Neighborhood Realtors" or similar territorial language unlocked once the data has built credibility — pending NAR + Real Broker compliance verification before use.

## Touch #2 (July 2026) test plan

Flip orientation from portrait to landscape on all three farms, hold everything else constant. Compare QR scan rate touch #1 vs touch #2 per farm. Three within-subject comparisons. Whichever orientation wins becomes the design system for touches 3-12.

## Approval workflow

1. Brian reviews PDF previews against his ground knowledge of each neighborhood ← **currently here**
2. Address any data feel checks (do the recent sales addresses match what Brian knows?)
3. Optional: swap hero photos if real ones are available
4. Final QR scan test on physical Geosential proof
5. Approve, schedule June 1 drop

## Reference

- Build scripts: `/home/claude/postcards/build_cards_final.py`, `heroes_final.py`, `build_pdfs.py`
- Print spec doc: `marketing/farms/june-2026-introduction-print-spec.md`
- Supabase project: `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS)
- Vendor: Geosential LITE ($47/mo, Mailbox Power underneath), upload at print.geosential.com
- Lead time: ~7 days, upload by May 22 for June 1 drop
