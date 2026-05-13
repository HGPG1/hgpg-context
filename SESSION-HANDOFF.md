<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — ReZEN review request backfill ✅ shipped

### What got built

Goal: hijack ReZEN's API to pull every closed deal from the last 24 months, push past clients into FUB as a tagged cohort for a review-request campaign (Viktor to build the actual FUB Automation 2.0 to send the asks).

#### ReZEN API
- **Auth header**: `X-API-Key: <key>` — NOT Bearer despite what the Bolt docs imply
- **Brian's yentaId**: `6eae7937-ba23-454e-92d5-3b556662f4c9`
- **Working endpoints**:
  - `GET https://yenta.therealbrokerage.com/api/v1/agents/me`
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/participant/{yentaId}/transactions/CLOSED?pageSize=N&pageNumber=N` — buyer-side
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/participant/{yentaId}/listing-transactions/CLOSED?pageSize=N&pageNumber=N` — listing-side
  - `GET https://arrakis.therealbrokerage.com/api/v1/transactions/{txId}` — full detail with participants
- **Closing date field**: `skySlopeActualClosingDate` (fallback `closedAt`, `rezenClosedAt`, `agentReportedClosingDate`)
- **Client filter**: `participants[].participantRole` in (`BUYER`, `SELLER`) AND `external: true`

#### Probe scripts (kept in iCloud)
Location: `~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/rezen-probe/`
- `probe.js`, `probe-auth.js`, `probe2.js`, `rezen-backfill.js`
- `.env` contains `REZEN_API_KEY`

#### Backfill results
- 138 buyer-side closed deals → 88 in 24-month window → 122 client rows
- 66 listing-side closed deals → 43 in 24-month window → 56 client rows
- **178 total client rows** imported to Supabase staging table

#### Supabase staging table
- `public.rezen_review_candidates` in HGPG Core (`ioypqogunwsoucgsnmla`)
- Status lifecycle: `pending_check` → `exists_in_fub` / `not_in_fub` → `pushed` / `failed` / `excluded`
- All 178 rows imported via MCP

#### TM page shipped to production
Files at commit `1888f69` on `main`:
- `lib/rezen-review/fub.ts` — FUB client (checkDuplicate, create, update, note, full row push)
- `app/api/rezen-review/route.ts` — GET (list+filter), PATCH (exclude/restore), POST (chunked push)
- `app/rezen-review/page.tsx` — server shell
- `app/rezen-review/client.tsx` — interactive UI

Production URL: `https://closings.homegrownpropertygroup.com/rezen-review`

#### Final campaign state
- **43 rows pushed to FUB** (clean — 33 net new past clients created, 10 updates to existing FUB people)
- **135 rows excluded** (dupes or team-agent deals not Brian-repped)
- 0 errors across all pushes

Tags applied to all 43:
- `review-request-2026-05` (cohort)
- `past-client-buyer` or `past-client-seller` (side)
- Stage set to `Past Client`
- Note added with closing details
- Address: buyer side only, never overwrote existing FUB addresses

### Pickup notes for next session

**Open work:**
1. **Hand cohort tag `review-request-2026-05` to Viktor** to build the FUB Automation 2.0:
   - Trigger on tag
   - Branch on side tag: `past-client-buyer` → Google review link; `past-client-seller` → Zillow review link
   - Stagger sends 5-10/day over 1-2 weeks (avoid clustered reviews)
2. **For future quarterly review pushes**: ReZEN API key + auth pattern + endpoint discovery is all documented. To repeat the campaign next quarter:
   - Re-run `rezen-backfill.js` to get fresh 24-month window
   - Bulk-insert into `rezen_review_candidates` with NEW cohort tag (e.g. `review-request-2026-q3`)
   - Use the existing `/rezen-review` page to triage and push

**Decisions captured in code:**
- BUYER side: property as home address (only on new creates)
- LISTING side: no address (rely on existing FUB address)
- Notes added both sides as audit trail
- Existing FUB people: tags + stage additive, address never overwritten
- Multi-client deals: each spouse/co-buyer is its own row

### Lessons learned

- **ReZEN docs lie about Bearer auth.** It's `X-API-Key`. Adding to known auth patterns.
- **iCloud-synced git repos can stall git commands by 10-30s.** Long-term, move `hgpg-transaction-manager` out of `~/Documents` (iCloud-synced) to `~/Code` (local-only) to avoid this.
- **TypeScript strict mode** in TM means all route handlers and React components need explicit types — no implicit `any`. First three Vercel builds failed on this; final commit added full types.
- **FUB API quirk**: `addresses[].type` should be lowercase (`'home'`, `'work'`). FUB UI shows it capitalized but API expects lowercase.

### Files for the brain

Add a project file at `projects/rezen-review-backfill.md` summarizing this campaign for future archival reference. The TM project file should add a section noting the new `/rezen-review` internal page exists.

### Infra changes

- New table `public.rezen_review_candidates` in HGPG Core (`ioypqogunwsoucgsnmla`) — staging only, can be dropped after Viktor's automation runs the campaign
- No schema changes to existing TM tables
