<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — ReZEN review request backfill 🟡 in progress

### What got built

Goal: hijack ReZEN's API to pull every closed deal from the last 24 months, push the clients into FUB as a tagged cohort for a review-request campaign (Viktor will build the actual FUB Automation 2.0 to send the asks).

#### ReZEN API discovered
- **Auth header**: `X-API-Key: <key>` — NOT Bearer despite what the Bolt docs imply
- **Brian's yentaId**: `6eae7937-ba23-454e-92d5-3b556662f4c9`
- **Working endpoints**:
  - `GET https://yenta.therealbrokerage.com/api/v1/agents/me` — auth verify + profile
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/participant/{yentaId}/transactions/CLOSED?pageSize=N&pageNumber=N` — buyer-side closed deals (138 total all-time)
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/participant/{yentaId}/listing-transactions/CLOSED?pageSize=N&pageNumber=N` — listing-side closed deals (66 total all-time)
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/{txId}` — full detail with participants array
- **Date filter field**: `skySlopeActualClosingDate` first, fallback `closedAt`, `rezenClosedAt`, `agentReportedClosingDate`. Filter for last 24 months client-side after fetching.
- **Client identification**: `participants[].participantRole` in (`BUYER`, `SELLER`) AND `external: true`. Other roles to filter out: `BUYERS_AGENT`, `SELLERS_AGENT`, `REFERRING_AGENT`, `REAL`, `BUYERS_LAWYER`, etc.
- **Multi-client deals**: one transaction can have N buyers or N sellers (spousal pairs, trusts, etc.) — we treat each as its own row, all sharing the same `rezen_transaction_id`.

#### Probe scripts (local, kept in iCloud)
Location: `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/rezen-probe/`
- `probe.js` — initial Bearer attempt (failed with 403)
- `probe-auth.js` — tries 8 auth header formats; found `X-API-Key` wins
- `probe2.js` — confirms transaction list + detail endpoints work
- `rezen-backfill.js` — paginated production pull, last 24 months, writes `rezen-backfill.csv` + `rezen-backfill.json`
- `.env` contains `REZEN_API_KEY`

#### Backfill results (last 24 months)
- **138 buyer-side closed deals** → 88 in the 24-month window → 122 client rows
- **66 listing-side closed deals** → 43 in the 24-month window → 56 client rows
- **178 total client rows**
- 93 with email+phone (high-value)
- 13 with email only
- 72 with neither email nor phone (name + address only)
- 7 deals returned no client participant from API (rentals/admin commissions like `b5a42f88` Lexington $2,550 — correctly tagged in `notes` column)

#### Supabase staging table (HGPG Core, project `ioypqogunwsoucgsnmla`)
- New table: `public.rezen_review_candidates`
- Status lifecycle: `pending_check` → `exists_in_fub` / `not_in_fub` → `pushed` / `failed` / `excluded`
- Unique index on `(rezen_transaction_id, coalesce(email, ''), coalesce(first||last, ''))` to prevent duplicate imports
- RLS enabled, service-role-only policy (TM uses service role for internal pages)
- All 178 rows imported via Supabase MCP (no service-role key needed to leave the system)

#### TM page + API route — files ready, NOT pushed yet
Local in this session output:
- `lib/rezen-review/fub.ts` — FUB client (checkDuplicate, create, update, note, full row push)
- `app/api/rezen-review/route.ts` — GET (list+filter), PATCH (exclude/restore), POST (chunked push)
- `app/rezen-review/page.tsx` — server shell
- `app/rezen-review/client.tsx` — interactive UI: filters, bulk select, push button, status badges
- `README.md` — placement + branch + merge instructions

Branch plan: `rezen-review-backfill` on `HGPG1/hgpg-transaction-manager`. Brian to push from Mac via gh CLI.

#### FUB push behavior (per row)
1. checkDuplicate by email → phone → exact-name match
2. If existing: PUT `/people/{id}` with merged tags + stage='Past Client' (additive, doesn't overwrite address)
3. If new: POST `/people` with tags + stage + emails + phones + addresses (BUYER side only)
4. Always: POST `/notes` with "Closed on [address] on [date] - [side], [price] (ReZEN [code])"
5. Chunks of 25, 2-second delay between chunks

#### Tags applied
- `review-request-2026-05` (cohort, all rows)
- `past-client-buyer` (buyer side rows only)
- `past-client-seller` (listing side rows only)

### Pickup notes for next session

**To finish the build:**
1. Brian creates branch `rezen-review-backfill` on `hgpg-transaction-manager`
2. Drop the 4 files from session outputs into the placements above
3. `git add` + commit + push from Mac
4. Verify Vercel preview deploys cleanly
5. Hit preview URL `/rezen-review`, sanity check the table renders 178 rows
6. Try pushing 3-5 high-value rows to FUB on the preview as a smoke test
7. Verify in FUB those people got the cohort tag + side tag + Past Client stage + note
8. Merge to `main`
9. Triage in FUB — exclude unwanted rows from the staging table UI as needed
10. Hand off to Viktor to build the FUB Automation 2.0 that fires the actual review request to anyone with `review-request-2026-05` tag

**Watchouts for the FUB push:**
- The `addresses` field on FUB Person needs `type` to be `"home"` (lowercase). Code uses lowercase. Verify on first push that this lands correctly.
- The `stage` field needs to exactly match the stage name in FUB. We're using `'Past Client'` — confirm that's the exact stage name in your FUB account before pushing. If it's `'Past Clients'` or something different, edit `fub.ts` line ~143 (the `stage:` value in `buildPersonUpdate` and `buildPersonCreate`).
- FUB rate limits aren't documented but historically ~250 requests/minute. 25 rows × 3 calls each (check + create/update + note) = 75 calls per chunk. Should be well under.

**Decisions captured in code:**
- BUYER side gets property as home address (assumption: they live there now)
- LISTING side gets no address (they sold and moved; rely on existing FUB address if any)
- Notes are added for both sides as audit trail
- Existing FUB people: tags + stage are additive/upgraded, address is never overwritten
- Multi-client deals: each spouse/co-buyer is its own row, gets its own FUB person

### Deferred / not built

- **No FUB Automation 2.0** — Viktor's job. Send copy will reference the `past-client-buyer` vs `past-client-seller` tag for buyer vs seller review URL (Google for buyer, Zillow for seller per existing TM convention).
- **No automation to re-pull ReZEN incrementally** — this is a one-shot backfill. New closed deals coming through TM will use the existing post-close FUB handoff (`lib/fubPostClose.ts`) and won't go through this staging table.
- **API key rotation** — service role key for HGPG Core wasn't exposed in chat (used MCP). ReZEN API key only lives in Brian's local `.env`. No rotation needed unless that file leaks.

### Project status updates

- `projects/transaction-manager.md` — add a "ReZEN backfill page" section (new internal page at `/rezen-review`)
- New entry recommended: `projects/rezen-review-backfill.md` documenting the one-shot campaign for future archival

### Infra changes that affect other apps

None today. Staging table is isolated, no schema changes to existing TM tables.
