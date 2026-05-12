<!-- Last Updated: 2026-05-12 -->

# Buyer Alerts

## Core Identity

- **Status:** 🟢 Session 1 shipped to branch (env vars + smoke test pending in production)
- **Lives inside:** Team Dashboard (`HGPG1/hgpg-team-dash`, route `/buyers`)
- **Live URL (target):** `team.homegrownpropertygroup.com/buyers`
- **Supabase project:** `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS) — shared with Team Dashboard
- **Cross-project lookups:** `ioypqogunwsoucgsnmla` (HGPG Core) for `team_members.phone`
- **Vercel project:** `prj_Q4dFcHGUvmUbUPaDydcHhdS2Laxd` (existing `hgpg-team-dash`)
- **Cron:** Vercel cron `*/30 * * * *` against `/api/cron/buyer-alerts`

## What it does

Buyer Alerts is the first piece of the MLS Dashboard Suite framing. An agent describes a buyer's wishlist in natural language; Claude Sonnet 4.6 parses it into a structured criteria JSON; a 30-minute cron scans `mls_property` for new or recently-modified Active matches and sends one short iMessage per new match to the owner's phone via LoopMessage. Repeat scans never re-alert thanks to a unique index on `(criteria_id, listing_key)`.

## Database schema (project `wdheejgmrqzqxvgjvfee`)

Migration `buyer_alerts_schema` (applied 2026-05-12):

- `public.buyer_criteria` — one row per buyer. Owner-scoped by `auth.uid()`.
  - `id`, `owner_user_id`, `buyer_label`, `raw_input`, `parsed` (jsonb), `status` (`active|paused|archived`), `created_at`, `updated_at`
  - Index: `(owner_user_id, status, created_at DESC)`
- `public.buyer_alerts` — one row per matched listing per buyer.
  - `id`, `criteria_id`, `listing_id`, `listing_key`, `matched_at`, `delivery_status` (`pending|sent|failed|skipped`), `delivered_at`, `delivery_error`, `match_reason` (jsonb)
  - Unique index: `(criteria_id, listing_key)` — the dedupe spine
  - Index: `(criteria_id, matched_at DESC)`
- RLS enabled on both. Policies:
  - `criteria_owner_all` — full CRUD scoped by owner
  - `alerts_owner_select` — SELECT only, joined through `buyer_criteria.owner_user_id`

## ParsedCriteria contract

LLM output and storage shape. Validated server-side; invalid fields coerce to null.

```jsonc
{
  "property_types": ["Single Family"] | null,   // advisory, not filtered in S1
  "min_price": 350000 | null,
  "max_price": 600000 | null,
  "min_bedrooms": 3 | null,
  "min_bathrooms": 2 | null,
  "min_sqft": 1800 | null,
  "max_sqft": null | null,
  "zips": ["28277","29707"] | null,
  "cities": ["Indian Land"] | null,             // case-insensitive match
  "subdivisions": ["Sun City Carolina Lakes"] | null,  // ILIKE
  "must_have_features": ["pool"] | null,        // advisory, not filtered in S1
  "must_avoid": ["55+","HOA above 200"] | null, // advisory, not filtered in S1
  "year_built_min": 2010 | null,
  "lot_size_min_acres": 0.5 | null,
  "max_dom": null | null,                        // advisory, not filtered in S1
  "notes": "free-form" | null
}
```

## Matcher rules (Session 1)

Filters applied server-side against `mls_property`:

- `mlg_can_view = true` (always, for index usage)
- `standard_status = 'Active'` (only active listings alert)
- `modification_timestamp >= now() - 24h` (recently new/changed only)
- `list_price` between `min_price` and `max_price` if set
- `bedrooms_total >= min_bedrooms` if set
- `bathrooms_total_integer >= min_bathrooms` if set
- `living_area` between `min_sqft` and `max_sqft` if set
- `postal_code = ANY(zips)` if set
- `city ILIKE` member of `cities` if set (case-insensitive client-side filter after fetch)
- `subdivision_name ILIKE ANY(subdivisions)` if set
- `year_built >= year_built_min` if set
- `lot_size_acres >= lot_size_min_acres` if set
- Sort `modification_timestamp DESC`, limit 50

Advisory fields (`property_types`, `must_have_features`, `must_avoid`, `max_dom`) are surfaced in `match_reason` but do not filter. Free-text feature matching is a Session 2 problem.

## Route map

| Route | Method | Purpose |
|---|---|---|
| `/buyers` | GET | List of current user's buyers + status controls |
| `/buyers/new` | GET | Two-step compose → parse-preview → save form |
| `/buyers/[id]` | GET | Detail: parsed criteria + recent alerts + "Run match now" |
| `/api/buyers/parse` | POST | LLM parse-only (preview round trip) |
| `/api/buyers/create` | POST | Parse + insert criteria |
| `/api/buyers/[id]/status` | PATCH | Set status (`active|paused|archived`) |
| `/api/buyers/[id]/match` | POST | Run matcher, insert alerts. Does NOT notify. |
| `/api/cron/buyer-alerts` | GET | Cron entrypoint — match + dedupe + LoopMessage send |

## Env vars (Vercel project `hgpg-team-dash`)

| Var | Status | Source |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | ✅ already set | existing |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | ✅ already set | existing |
| `TEAM_EMAILS` | ✅ already set | existing |
| `SUPABASE_SERVICE_ROLE_KEY` | ⏳ MUST ADD | Supabase dashboard → wdheejgmrqzqxvgjvfee → API |
| `ANTHROPIC_API_KEY` | ⏳ MUST ADD | Brian's Anthropic console |
| `HGPG_CORE_SUPABASE_URL` | ⏳ MUST ADD | `https://ioypqogunwsoucgsnmla.supabase.co` |
| `HGPG_CORE_SUPABASE_SERVICE_ROLE_KEY` | ⏳ MUST ADD | Supabase dashboard → ioypqogunwsoucgsnmla → API |
| `LOOPMESSAGE_API_KEY` | ⏳ MUST ADD | Match TM repo var name + value |
| `LOOPMESSAGE_SECRET` | ⏳ MUST ADD | Match TM repo var name + value |
| `CRON_SECRET` | optional | Used only when not invoked by Vercel cron (auth fallback for manual triggers) |

The Vercel MCP server exposed to this session does not include env-var write tools (only listing/deployments/logs/toolbar), so these must be added by hand in the Vercel dashboard. Once they land, Vercel auto-redeploys on the next push to `main` (or trigger a redeploy from the dashboard).

## LLM parser

- Model: `claude-sonnet-4-6`
- API: Anthropic Messages, `2023-06-01` version
- System prompt: forces JSON-only output, no fences, no prose
- Retry once with stricter system message on first failure
- Final fallback: return all-null criteria with raw input stored in `notes`

## iMessage delivery

- Channel: LoopMessage (Twilio is deprecated — explicitly the only iMessage path)
- Phone lookup: `team_members.phone` on HGPG Core, matched by buyer owner's email (resolved via `auth.admin.getUserById` on Listing Reports + MLS service role, then `ilike` on `team_members.email`)
- Body: one short iMessage per match — buyer label, address, beds/baths/sqft/price, days listed, deep link to Canopy MLS
- No phone on file → alert row is created with `delivery_status='skipped'` and `delivery_error='no phone on file'`

## API surface (lib)

`src/lib/buyerAlerts.ts` exports:

- `ParsedCriteria`, `BuyerCriteriaRow`, `AlertRow`, `MatchCandidate`, `CriteriaStatus`
- `parseCriteriaWithLLM(rawInput)`
- `getBuyerCriteria(userId)`, `getBuyerCriterion(id)`, `createBuyerCriteria(input)`, `updateBuyerStatus(id, status)`
- `getRecentAlerts(criteriaId, limit?)`
- `findMatchesForCriteria(criteria, client?)`, `buildMatchReason(parsed, match)`
- `recordMatch(criteriaId, listing, reason, client?)` → `{ created: boolean }` (false on unique-conflict)
- `getServiceRoleClient()`, `getCoreServiceRoleClient()` — service-role helpers for the cron

## Acceptance criteria status

- ✅ DB schema applied with RLS
- ✅ Logged-in user can create a buyer with natural-language input (UI shipped)
- ✅ Parsed criteria render as bullets on detail page
- ✅ "Run match now" hits live MLS and inserts alerts
- ✅ Dedupe works (unique index on criteria_id+listing_key, `recordMatch` returns `{created:false}` on conflict)
- ✅ Cron registered in `vercel.json` at `*/30 * * * *`
- ⏳ Manual cron invocation sending real iMessage — requires LoopMessage + Anthropic + service role env vars (not exposed to MCP)
- ✅ Production build clean (`npx tsc --noEmit` + `npx next build`)
- ⏳ Production env vars set on Vercel — pending human action (see table above)

## Session 1 limitations (carried to Session 2 backlog)

- Parsed criteria preview is read-only; agent can't edit individual fields (must re-create from raw input)
- No agent management / sharing — a buyer is owned strictly by `auth.users.id`
- `must_have_features` / `must_avoid` / `property_types` are captured but not enforced server-side
- No quiet-hours or throttle — relies on the 24-hour modification window to avoid back-spam
- No "send me a test alert" button to validate LoopMessage path without waiting for real new inventory
- No alert history pagination beyond the 30 most recent on the detail page
- Email fallback when phone is missing — currently we just mark `delivery_status='skipped'`

## Pickup notes for Session 2

1. Set the 6 missing Vercel env vars (see table)
2. Trigger one manual cron run via `curl -H "Authorization: Bearer $CRON_SECRET" https://team.homegrownpropertygroup.com/api/cron/buyer-alerts` (or hit it via the Vercel cron schedule)
3. Create a smoke-test buyer with very loose criteria (e.g. "anywhere in Charlotte under $1.5M, 2+ bed") to guarantee a match within 30 minutes
4. Once delivery is confirmed end-to-end, file Session 2 work: criteria-edit UI, manual test-alert button, agent-management (assign a buyer to a teammate), property_type / must_avoid filtering

## Learnings

1. **Cross-project identity is by email, not by uid.** Listing Reports + MLS has its own `auth.users` table; HGPG Core has its own `team_members`. The join is `auth.users.email` ↔ `team_members.email`. The cron uses two service roles (one per project) to traverse.
2. **PostgREST `.or()` with `ilike` doesn't handle commas in patterns.** The matcher strips commas from subdivision names before composing the OR expression — case-insensitive subdivisions still work, but commas in canonical names would break parsing.
3. **Dedupe at the unique index is cleaner than dedupe in app code.** Inserting on conflict gives a single source of truth and avoids the read-modify-write race where two cron runs in flight could both decide "new."
4. **Service-role auth is mandatory for the cron.** Vercel cron has no user cookie, so RLS would block everything. `SUPABASE_SERVICE_ROLE_KEY` is non-negotiable.
5. **Anthropic JSON mode is a system-message convention, not a parameter.** Sonnet 4.6 follows the "JSON only" system message reliably on first try; the second-attempt retry is defensive insurance.
