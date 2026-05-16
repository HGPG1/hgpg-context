<!-- Last Updated: 2026-05-15 -->

# June 2026 Postcard Brief - Introduction Mailer

## Print specs

- **Vendor:** Mailbox Power (Pro membership)
- **Size:** 5.5 x 8.5 (jumbo, postcard rate)
- **Stock:** Standard gloss, full color both sides
- **Postage:** $0.78/piece (USPS jumbo postcard rate)
- **Quantity:** 1,345 total (Bent Creek 430, Bridgemill 600, Queensbridge 315)
- **Drop date:** June 1, 2026
- **Variants:** Three farm-specific designs sharing one design system

## Strategy

This is the introduction mailer. It does ONE job: establish recognition. Most homeowners in these farms have never heard of HGPG. This card is the first of 12 touches in Year 1 - the goal is not a lead, it's name recognition that compounds over the remaining 11 sends.

**Hierarchy:** Brian's face + name first, neighborhood second, data third, CTA fourth. Resist the urge to lead with a selling proposition. The data IS the value.

## Brand requirements (NON-NEGOTIABLE)

- **Canva brand kit:** kAHFKMi4Q7g
- **Colors:** #2A384C dark navy, #A0B2C2 steel blue, #D1D9DF light steel, #F0F0F0 off-white. NO GREEN.
- **Fonts:** Cooper Hewitt (body), Sansita Regular (display). NO Cormorant Garamond.
- **Tagline placement:** "Growth Starts Here, At the Roots." appears once, near the logo
- **No em dashes anywhere.** Use hyphens, commas, parentheses, or period breaks
- **Compliance footer:** Real Broker LLC, 7612 Charlotte Highway, Indian Land, SC 29707
- **Equal Housing logo** in footer, MLS logo optional

## Front of card (the show-stopper)

**Top third:** Hero headline in Sansita Regular, white on navy

    The [Neighborhood] Market,
    In Plain English.

**Middle third:** Brian's professional headshot (round crop, steel blue ring), positioned left. Right side carries name + title in Sansita, contact info in Cooper Hewitt:

    Brian McCarron
    Broker / Owner
    Home Grown Property Group
    
    803-902-3700
    brian@homegrownpropertygroup.com

**Bottom third:** Tagline in Sansita Regular small caps, centered, light steel on navy:

    GROWTH STARTS HERE, AT THE ROOTS.

## Back of card (the substance)

**Headline strip (top, navy band, white Sansita):**

    [Neighborhood] - Q1 2026 Market Snapshot

**Three stat tiles, light steel background, navy numbers, Cooper Hewitt labels:**

| Tile | Bent Creek | Bridgemill | Queensbridge |
|---|---|---|---|
| Median Sold | $785,000 | $819,990 | $724,500 |
| Sold in last 12mo | 19 homes | 19 homes | 10 homes |
| Sale-to-list ratio | 98.3% | 98.1% | 98.4% |

**Body copy (steel blue background, navy text, Cooper Hewitt, ~3 sentences):**

    I am Brian, and Home Grown Property Group is the real estate
    team based right here in Indian Land at 7612 Charlotte Highway.
    Over the next year you will see a monthly mailer from us with
    the data we wish more agents would actually share: what is
    selling in [Neighborhood], for how much, and what it means for
    your home's value. No sales pitch, just the numbers.

**QR + CTA block (bottom right, navy):**

    [QR code, ~1.25" square]
    
    See your home's value
    homegrownpropertygroup.com/farm/[slug]

QR URLs:

    Bent Creek:    https://www.homegrownpropertygroup.com/farm/bent-creek?c=bc-2026-06
    Bridgemill:    https://www.homegrownpropertygroup.com/farm/bridgemill?c=bm-2026-06
    Queensbridge:  https://www.homegrownpropertygroup.com/farm/queensbridge?c=qb-2026-06

**Compliance footer strip (off-white, small Cooper Hewitt, navy text):**

    Home Grown Property Group | A team within Real Broker LLC
    7612 Charlotte Highway, Indian Land, SC 29707 | 803-902-3700
    [Equal Housing icon] [MLS icon]

## What NOT to put on this card

- "Just listed" or "Just sold" anything (wrong month, comes in August)
- "Thinking of selling?" or any direct CTA to list (too aggressive on touch 1)
- Stock photos of generic houses (only the recipient's home matters, and that comes via Mailbox Power's Street View merge if enabled)
- Multiple QR codes (one QR, one job)
- Phone number larger than the QR (we want them online, not calling)
- Any green (brand violation)

## Mailbox Power-specific setup

When uploading to Mailbox Power:

1. Upload three separate designs as 5.5x8.5 templates (one per farm)
2. Tag each with the farm name in their design library
3. Enable dynamic QR with text alert routing to Brian's phone (803-902-3700)
4. If using their Street View merge feature, enable it for the BACK of the card only - it gets a thumbnail of each recipient's home next to the body copy, replacing the generic stat tile if necessary. Decide whether to use this for June or hold for July depending on how it looks in proof.
5. Upload the three address CSVs from iCloud workspace:
   - farm-addresses-bent-creek.csv (430 homes)
   - farm-addresses-bridgemill.csv (600 homes)
   - farm-addresses-queensbridge.csv (315 homes)

## Tracking setup (Supabase, do BEFORE drop)

Run this in HGPG Core to log the three June campaigns:

    INSERT INTO farm_campaigns (neighborhood_id, qr_code_slug, send_date, theme, format, vendor, quantity, design_notes)
    VALUES
      ('4f3b0cbf-8009-4573-a130-c1df11a91a28', 'bc-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'mailbox-power', 430, 'Year 1 Touch 1 - introduction with Q1 stats'),
      ('d8983d8c-8004-4f41-8cae-367f013340d7', 'bm-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'mailbox-power', 600, 'Year 1 Touch 1 - introduction with Q1 stats'),
      ('5114a40e-1bc4-4701-9451-64a12b575086', 'qb-2026-06', '2026-06-01', 'introduction', '5.5x8.5 jumbo', 'mailbox-power', 315, 'Year 1 Touch 1 - introduction with Q1 stats');

After insert, run:

    SELECT name, qr_code_slug FROM farm_campaigns
    JOIN farm_neighborhoods ON farm_neighborhoods.id = farm_campaigns.neighborhood_id
    WHERE send_date = '2026-06-01';

Use the returned qr_code_slug values as the `?c=` parameter on each farm's QR.

## Approval workflow

1. Brian (or Claude on Brian's behalf) drafts in Canva using brand kit kAHFKMi4Q7g
2. Three variants exported as PDF/X-1a for print
3. Compliance check: address, broker line, Equal Housing logo present and legible
4. Upload to Mailbox Power, request proof (turnaround ~24 hrs)
5. Approve proofs, schedule June 1 drop
6. Log campaigns to Supabase (SQL above)
7. Mailers drop, scans start logging to farm_leads automatically

## Budget for June

| Line | Amount |
|---|---|
| Mailbox Power Pro (annual, $990) | $82.50 amortized to June |
| Postage 1,345 x $0.78 | $1,049 |
| Postcards (free, included in membership) | $0 |
| Canva design time (in-house) | $0 |
| **Total June outlay** | **$1,131** |

## What success looks like for this drop

- ~13-27 QR scans (1-2% scan rate is typical for first-touch direct mail)
- 2-5 form submissions on /farm landing pages
- 0-2 phone calls / direct text replies
- Zero closings (this is touch 1 of 12 - the math compounds at month 4-6)

If you see more than 1 form lead per farm in week 1, that's a strong signal. Below that, normal and expected.

## Next month preview

July 2026 is the Q2 Market Update - same design system, different headline ("[Neighborhood] Q2 - What Changed"), refresh the three stat tiles with Q2 data, same QR/CTA structure. We will reuse this brief as a template.
