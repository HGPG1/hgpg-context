# HGPG - Ads Project Handoff: June 2026 Geo-Farming Postcards

## What you're being asked to do

Build three print-ready postcard PDFs for the June 1, 2026 mailer drop of HGPG's geographic farming program. Each postcard is a 5.5 x 8.5 inch jumbo, front and back, with one neighborhood-specific variant per farm: **Bent Creek, Bridgemill, Queensbridge**.

The QR codes route to real landing pages already live on homegrownpropertygroup.com. Each has campaign attribution baked in. The codes need to be scannable, not placeholders.

## Project context

This is touch #1 of a 12-month farming program. The goal of THIS card is recognition, not selling. It introduces Brian + HGPG to homeowners who have never heard of him, and establishes "you'll see one of these every month with real market data, no sales pitch." Touches 2-12 build on this foundation.

Vendor: Geosential LITE ($47/mo, Mailbox Power underneath). They accept custom PDF uploads, so we're not template-locked. Final outputs upload to print.geosential.com.

## Brand requirements (NON-NEGOTIABLE)

- **Colors only**: navy #2A384C primary, steel blue #A0B2C2 secondary, light steel #D1D9DF tertiary, off-white #F0F0F0 background. NO green. NO red. NO bright accents.
- **Fonts**: Sansita Regular for display/headlines, Cooper Hewitt for body and labels. NO Cormorant Garamond. NO serifs other than Sansita.
- **Tagline**: "Growth Starts Here, At the Roots." appears once, near the logo
- **No em dashes**. Hyphens, commas, parentheses, or period breaks only.
- **Compliance footer required**: Real Broker LLC, 7612 Charlotte Highway, Indian Land, SC 29707
- **HGPG logo must appear on both sides** of the card
- Canva brand kit ID: `kAHFKMi4Q7g`

## Print specs

- Size: 5.5 x 8.5 inches (jumbo postcard, USPS jumbo rate)
- Bleed: 0.125 inch on all sides (final cut size 5.5 x 8.5)
- Safe area: 0.25 inch inside trim
- Resolution: 300 DPI minimum
- Color: CMYK (Geosential will convert if uploaded as sRGB, but CMYK is cleaner)
- Export: PDF/X-1a, one PDF per farm, front and back on separate pages or as separate files
- Stock: standard glossy, full color both sides (Geosential default)

## Front of card layout

Top 55%: Hero photo (different per farm, see below)

Below photo, navy band: headline in Sansita Regular white text:
> "The [Neighborhood] Market, In Plain English."

Below headline, smaller Cooper Hewitt subhead in steel blue:
> "A monthly market update from your neighbors at Home Grown Property Group"

Bottom strip:
- HGPG logo centered
- Below logo, tagline "GROWTH STARTS HERE, AT THE ROOTS." in Sansita Regular small caps light steel

## Back of card layout

Top navy header band: Sansita Regular white text:
> "[Neighborhood] - Q1 2026 Market Snapshot"

Three stat tiles in a horizontal row, light steel #D1D9DF background, navy Sansita Regular numbers, Cooper Hewitt small labels below each number:

| Bent Creek | Bridgemill | Queensbridge |
|---|---|---|
| $785,000 / Median Sold Price | $819,990 / Median Sold Price | $724,500 / Median Sold Price |
| 19 / Homes Sold Last 12 Months | 19 / Homes Sold Last 12 Months | 10 / Homes Sold Last 12 Months |
| 98.3% / Sale-to-List Ratio | 98.1% / Sale-to-List Ratio | 98.4% / Sale-to-List Ratio |

Body copy block, steel blue #A0B2C2 background, navy Cooper Hewitt text, three sentences (substitute neighborhood name):
> "I am Brian, and Home Grown Property Group is the real estate team based right here in Indian Land at 7612 Charlotte Highway. Over the next year you will see a monthly mailer from us with the data we wish more agents would share: what is selling in [Neighborhood], for how much, and what it means for your home's value. No sales pitch, just the numbers."

Two-column footer area:
- **LEFT COLUMN**: Round headshot of Brian (steel blue ring border) + name and contact stacked next to it:
    - "Brian McCarron" Sansita Regular navy
    - "Broker / Owner" Cooper Hewitt small
    - "803-902-3700" Cooper Hewitt
    - "brian@homegrownpropertygroup.com" Cooper Hewitt
- **RIGHT COLUMN**: QR code (1.25 inch square) with caption above "See your home's value" Cooper Hewitt and URL below in small Cooper Hewitt

Compliance footer strip at very bottom, off-white #F0F0F0 background, navy Cooper Hewitt small text:
> "Home Grown Property Group | A team within Real Broker LLC | 7612 Charlotte Highway, Indian Land, SC 29707"

Equal Housing Opportunity icon in footer right corner.

## Per-farm variables (the three variants)

### Bent Creek (29720 Lancaster)

- Headline: "The Bent Creek Market, In Plain English."
- Stats: $785,000 / 19 / 98.3%
- QR URL: `https://www.homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06`
- URL caption text: `homegrownpropertygroup.com/farm/bent-creek`
- Hero photo: Bent Creek front gate (the entry experience is the local recognition trigger). v1: stock gated-community entrance, v2: real Bent Creek gate photo Brian will source.
- Print quantity: 430

### Bridgemill (29707 Indian Land)

- Headline: "The Bridgemill Market, In Plain English."
- Stats: $819,990 / 19 / 98.1%
- QR URL: `https://www.homegrownpropertygroup.com/farm/bridgemill?c=bm-2026-06`
- URL caption text: `homegrownpropertygroup.com/farm/bridgemill`
- Hero photo: Bridgemill clubhouse + golf course (the iconic landmark of the neighborhood). v1: real Bridgemill clubhouse photo (sourced via web search, copyright OK for v1 internal review only, swap before print). v2: confirm rights or have Brian shoot.
- Print quantity: 600

### Queensbridge (29707 Indian Land)

- Headline: "The Queensbridge Market, In Plain English."
- Stats: $724,500 / 10 / 98.4%
- QR URL: `https://www.homegrownpropertygroup.com/farm/queensbridge?c=qb-2026-06`
- URL caption text: `homegrownpropertygroup.com/farm/queensbridge`
- Hero photo: Pulte Worthington plan home or other typical Queensbridge home exterior. v1: stock craftsman porch / suburban home, v2: real Queensbridge home photo.
- Print quantity: 315

## QR code generation requirements

Use Python `qrcode` library or equivalent. Each QR must:

- Encode the exact URL (with `?c=` parameter) for that farm
- Render at 1.25 inch square minimum at 300 DPI (375x375 pixels)
- Have a quiet zone (white margin) of at least 4 modules around the code
- Use high error correction (ECC level H) so a logo can be embedded in the center if desired
- Be solid navy #2A384C on white background (NOT pure black - matches brand)
- Verify each code scans correctly using a QR reader before exporting the PDF

## Logo

HGPG logo lives in:
- Canva brand kit `kAHFKMi4Q7g` (preferred for Canva-based workflow)
- Likely also on homegrownpropertygroup.com header (downloadable as image)
- Brian can upload directly if needed

Place logo on:
- Front bottom strip, centered, white or light steel version against navy background
- Back compliance footer area, small, navy version against off-white background

## What NOT to put on this card

- "Just Listed" or "Just Sold" anything
- "Thinking of selling?" or any direct CTA to list
- Generic stock photos of unrelated houses (placeholder OK, but mark for swap)
- Multiple QR codes
- Phone number larger than the QR
- Any green color, any red color, any em dashes
- Cormorant Garamond or any serif other than Sansita Regular

## Deliverables

1. Three print-ready PDFs (one per farm), 5.5 x 8.5 with bleed, 300 DPI, PDF/X-1a
2. Three scannable QR codes verified before embed
3. Source files (Figma / Photoshop / Canva / whatever tool) for monthly variant production going forward (July through May 2027 will reuse this design system with refreshed data and themes)
4. Screenshot or thumbnail of each card for Brian to approve before final export
5. Final PDFs uploaded to brain repo at `marketing/farms/june-2026-introduction/` folder

## Approval workflow

1. Designer drafts v1 with placeholder photos and real QR codes
2. Brian reviews via screenshot, requests revisions
3. Designer applies revisions, swaps in final photos if Brian provides them
4. Compliance check: logo on both sides, broker line + address present and legible, Equal Housing icon, QR scans correctly to the right URL
5. Approve final, export PDF/X-1a
6. Upload to Geosential, request proof, schedule June 1 drop

## Reference files in brain repo

- `marketing/farms/june-2026-introduction.md` - full brief
- `marketing/geo-farming.md` - program plan and vendor decisions
- `brand.md` - brand standards (colors, fonts, logo usage)

## Timeline

- Design system v1 ready for Brian review: ASAP, target within 3 days
- Final approved PDFs: by May 22, 2026 (Geosential needs ~7 days lead time for June 1 drop)
- Photos can be swapped between v1 and final if Brian sources real ones in that window
