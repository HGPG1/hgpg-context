<!-- Last Updated: 2026-05-08 -->

# Team Dashboard

**Status:** 🟡 BUILDING (Tab 1 in progress 2026-05-08)
**Live URL:** TBD (proposed `team.homegrownpropertygroup.com`)
**Repo:** `HGPG1/hgpg-team-dash` (to be created)
**Supabase:** `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS — same project that holds `mls_property`, `listings`, `showings`, `weekly_stats`)
**Vercel team:** `team_FietQPKCmnyioG2n0FdteQCV`

## Purpose

Team-facing dashboard that pairs two tools onto a single auth-gated surface:

- **Tab 1 — Inventory + Showings.** Live HGPG team listings (active, under contract, pending, recently closed) with showing activity, ListTrac platform views, and feedback rolled up per property.
- **Tab 2 — Market Intelligence.** Neighborhood snapshot picker (zip / subdivision / radius) with median price, DOM, sold volume, list-to-sold ratio, months-of-supply, and 30-day movers list.

Both tabs read from the same MLS Grid replication that powers the CMA Engine. The infrastructure is already paid for, so each new tool on top is cheap marginal value. Tab 1 is the entry point because it shows the team something they look at every day (their own listings).

This is **NOT** the same as the Listing Report Portal (`reports.homegrownpropertygroup.com`) — that's seller-facing, this is internal team-facing. It is also NOT the same as `dash.homegrownpropertygroup.com` — that stays Signature-only.

## Audience

Brian, Ashley, Brenda, Taylor (agents). Lauren and Vessie are also on the office key — Lauren handles closings and Vessie's role is unclear. Inventory tab unconditionally shows everything tied to office key `CAR118249224` so anything they list shows up automatically.

Don is a TC, not an agent — he uses TM and TC Concierge instead. Don shows as `Inactive` on `mls_member` which is fine.

## Data backbone

Office key: **`CAR118249224`** (NOT R04075 — that's a Canopy display ID that does not appear in the MLS Grid feed).

Single filter `mls_property.list_office_key = 'CAR118249224'` returns the team's full listing history. Verified counts at build time: 42 closed, 14 canceled, 3 active, 2 expired, 1 active under contract, 1 pending.

### Critical join semantics

| Source table | Key | Notes |
|---|---|---|
| `mls_property` | filter on `list_office_key = 'CAR118249224'` | live MLS Grid feed, refreshes every 15 min |
| `listings` (manual portal table) | join on `listings.mls_number = REPLACE(mls_property.listing_id, 'CAR', '')` | NOT every team listing has a portal row — only ones that were onboarded into the Listing Report Portal |
| `showings` | `showings.listing_id = listings.id` | only available for listings that have a `listings` row |
| `weekly_stats` | `weekly_stats.listing_id = listings.id` | ListTrac platform views, only on listings with a portal row |
| `mls_media` | `mls_media.resource_record_key = mls_property.listing_key AND preferred_photo_yn = true` | photos NOT YET SYNCED — currently every row returns null. Hero photo falls back to `listings.hero_photo_url` then to a brand placeholder |

### Address composition

`mls_property.unparsed_address` is null for most rows. Build display address from:

    TRIM(CONCAT_WS(' ', street_number, street_name, street_suffix))

Then append `, city, state_or_province zip` when needed.

## Architecture

### Stack

- **Next.js 16 App Router** + Tailwind v4 (matches Brain App)
- **Supabase Auth** magic link, single-table allow-list of team emails
- **Recharts** for Tab 2 (already proven in TM)
- **Vercel** auto-deploy on push to main

### Auth model

Magic link to `@homegrownpropertygroup.com` emails only. Allow-list stored in env var `TEAM_EMAILS`, comma-separated. No PIN — magic link is the auth gate. iOS keychain handles persistence.

### Routes

    /                         redirect to /inventory
    /auth/login               magic link request
    /auth/callback            Supabase auth callback
    /inventory                Tab 1 — listings grid
    /inventory/[listingId]    Tab 1 drilldown
    /market                   Tab 2 — neighborhood picker
    /market/[zipOrArea]       Tab 2 results

### Data fetching

Server components query Supabase directly via `@supabase/supabase-js` with the anon key + RLS-style policy filtering by signed-in user email against the team allow-list. No service role key needed for read-only operations.

## Tab 1: Inventory + Showings

### Top-level grid

Cards rendered as a responsive grid (3 cols desktop, 2 tablet, 1 mobile). Each card shows:

- Hero photo (preferred MLS photo when available, falls back to portal hero, falls back to neutral placeholder)
- Address (composed from MLS parts)
- City + status pill (color-coded: navy = Active, signature brown = Active Under Contract, steel = Pending, dim = Closed/Canceled/Expired)
- List price
- DOM badge
- Listing agent display name (from `mls_member.member_full_name` joined on `list_agent_key`)

Default view filters to `Active`, `Active Under Contract`, `Pending`. Toggle pills above the grid for `Closed (last 90 days)`, `All time`, `Canceled/Expired`.

### Drilldown

Click any card → `/inventory/[listingId]`. Sections:

1. Hero strip with all the grid card info enlarged
2. **Showing activity** — chronological list, pulled from `showings` table when listing has a portal row. If no portal row, show "Not yet onboarded into the Listing Report Portal" with a button to add it.
3. **Platform views** — last 30-day breakdown from `weekly_stats.platform_breakdown` JSONB. Bar chart, top platform first.
4. **Feedback** — concatenated `feedback_summary` from showings, sorted newest first, with star ratings.
5. **Key dates** — list date, price changes, status changes (from `mls_property.modification_timestamp`, `price_change_timestamp`, `pending_timestamp`, `contract_status_change_date`).

### Data freshness

MLS data refreshes every 15 minutes via the existing CMA Engine cron. Tab 1 reflects whatever the latest sync caught. Stale-warning banner if `mls_sync_state.last_synced_at` is more than 30 minutes old.

## Tab 2: Market Intelligence (NEXT SESSION)

Spec stub:

- Search bar accepts zip, subdivision name, or address (with radius slider)
- Recharts: median price line (12 months), DOM line (12 months), sold volume bars, list-to-sold ratio, months-of-supply
- Active inventory count + last-30-day movers (new pendings, new closings, price changes)
- All charts read from `mls_property` filtered by area
- Reuse Recharts components from TM where possible

## Build order

1. ✅ Verify office key and data shape (`CAR118249224`, 5 active/UC/pending)
2. 🔄 Scaffold repo + Vercel project
3. 🔄 Build auth (magic link + allow-list)
4. 🔄 Build Tab 1 grid
5. 🔄 Build Tab 1 drilldown
6. ⏭️ Build Tab 2 (next session)
7. ⏭️ DNS swap to `team.homegrownpropertygroup.com`

## Open follow-ups

- **Vessie Lopez** is on the office key but not on the agent roster in the brain. Confirm her role and whether she should appear as a listing agent in the dashboard.
- **MLS Grid Media sync** is parked. Until it runs, hero photos pull from the portal `listings.hero_photo_url` (only available for the ~7 listings already onboarded). Everything else falls back to placeholder.
- **Subdomain** — proposed `team.homegrownpropertygroup.com`. Alternatives: `office.homegrownpropertygroup.com`, `inside.homegrownpropertygroup.com`.

## Why this build helps

- Team always has a single page to glance at to see "what is HGPG selling right now and how is it performing"
- Showing activity in one place stops the every-Monday "what showings did we have last week" Slack dance
- Drives more traffic to the Listing Report Portal because team members will notice when a listing has no portal row and onboard it
- Reuses 100% of the MLS Grid pipeline that's already paid for and running
