<!-- Last Updated: 2026-05-15 -->
# Charlotte New Construction Guide

**Status:** 🟢 SHIPPED (production), incentive triage cadence established

## URLs
- Production: https://newconstruction.homegrownpropertygroup.com
- Repo: HGPG1/charlotte-new-construction-nextjs (Next.js)
- Vercel team: team_FietQPKCmnyioG2n0FdteQCV

## Data layer

**Supabase project:** HGPG Signature + Relocation (`fkxgdqfnowskflgbuxhm`) — NOT HGPG Core. Common gotcha.

**Tables:**
- `nc_builders` — builder roster (Lennar, DR Horton, M/I Homes, Empire, Meritage, David Weekley, Dream Finders, LGI, Mattamy, Toll Brothers, True Homes, etc.)
- `nc_incentives` — builder incentives with `is_active` flag, optional `expiration_date`, `incentive_type` enum (`rate_buydown`, `closing_costs`, `price_reduction`, `free_upgrades`, `other`)
- RLS: public read on `is_active = true` only

**Schema for `nc_incentives`:**
- `id` uuid
- `builder_id` uuid (FK to nc_builders)
- `incentive_type` text (required)
- `value_amount` integer (nullable)
- `description` text (required)
- `expiration_date` date (nullable)
- `is_active` boolean (required)
- `created_at`, `updated_at` timestamps

## Admin

- PIN-gated admin UI at `/admin` (default PIN 1847, env-overridable)
- API routes: `/api/admin/auth`, `/api/admin/builders`, `/api/admin/incentives` (POST/PATCH/DELETE all gated)
- Soft-deactivate by setting `is_active = false`, preserves history

## Incentive triage protocol

Incentives bloat fast because different sales reps push different language for the same underlying promo, and builders rotate offers in 1-2 week cycles. Run triage when active count creeps above ~40 or when a builder has 5+ active rows.

**Triage steps:**
1. Pull live state: `SELECT b.name, i.* FROM nc_incentives i JOIN nc_builders b ON b.id = i.builder_id WHERE i.is_active = true ORDER BY b.name`
2. Group by builder. Flag clusters of 3+ active rows on the same builder.
3. Identify dupes — same promo, different wording, often one with a specific expiration and one without.
4. Identify stale — descriptions referencing past months ("April 30th") with null `expiration_date` are the worst offenders.
5. Soft-deactivate (don't delete) — preserves audit trail for season-over-season analysis.

## Builder refresh cadences

These rates of change are battle-tested from the May 2026 triage:

| Builder | Cadence | Notes |
|---|---|---|
| **Lennar** | Weekly | "Dream Savings" promo URL `/promo/gallen_dsv26` rotates with new 7-day sign window. Specific rates pulled one week are stale the next. Use a generic placeholder ("Dream Savings: temporary buydown and closing cost credits, offers rotate weekly — call agent") rather than chasing specifics. |
| **Toll Brothers** | ~2 weeks | Standard 2/1 buydown at 3.99% first-year (6.22% APR). Same promo language every cycle, just rolling date windows. Safe to extend `expiration_date` 6-8 weeks out and refresh quarterly. Promo blankets all Charlotte Division communities (incl. Pines at Sugar Creek in Indian Land). |
| **DR Horton** | Monthly + community-specific | "Red Tag" events run ~3-4 weeks. Closing cost incentives ($9,500 with DHI Mortgage) are division-wide. Rate buydowns are community-specific and don't apply division-wide despite how reps phrase them. |
| **M/I Homes** | Stable named programs | "Great Low Rate" (4.875%), "Zero Down 2026", "Investment Home 5.875%", "2/1 Buydown" all have dedicated pages on mihomes.com and persist for months. Easy to keep accurate. |
| **Meritage** | Slow rotation | "Blue Homes" promo identifier signals their move-in-ready inventory program. |
| **Empire** | Community-specific | Multiple active rows for different communities (Brixton, Caswell, Equinox, Camellia Gardens, Harris Farms, Oak Creek, Calico Ridge) — these ARE distinct, not dupes. |

## May 15, 2026 triage outcome

Started at 79 total / 45 active. Ended at 77 total / 35 active.

**Deactivated (10 rows):**
- David Weekley: 1 duplicate 30K anniversary entry
- Dream Finders: 1 row with "April 30th" in body, null expiration
- Meritage: 1 duplicate 4.99% fixed rate entry
- DR Horton: 6 rows (stale May expirations superseded by June rolls, vague "available with builder lender" placeholders, speculative July rate promos)
- Lennar: 3 rows (5/22 trio — specific rate, $7K closing, $5K Flex — replaced with generic weekly-rotation placeholder)

**Added (4 rows):**
- M/I Homes: 5.875% investment home rate, Zero Down Program 2026
- Lennar: Generic Dream Savings placeholder
- DR Horton: Generic $9,500 closing costs with DHI Mortgage placeholder

**Updated (1 row):**
- Toll Brothers: extended through 6/30, added note about 2-week refresh cadence

## Open items

- **M/I `85cca8ac`** — 4.875% Great Low Rate expires 5/29 (2 weeks out). Verify with M/I rep whether it'll roll forward.
- **Meritage `3373880f`** — $3,000 May closing bonus expires 5/31.
- Consider an admin dashboard "Expiring This Week" widget so Brian can triage on-demand without waiting for the list to balloon.

