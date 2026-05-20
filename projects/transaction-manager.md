<!-- Last Updated: 2026-05-21 -->

# Transaction Manager

- **Status:** 🟡 Active build, ongoing Don feedback batches
- **URL:** closings.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-transaction-manager
- **Local path:** `~/Documents/hgpg-transaction-manager` (NOT `~/Projects` - older brain noted this incorrectly)
- **Vercel project ID:** `prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`
- **Vercel team:** `team_FietQPKCmnyioG2n0FdteQCV`
- **Supabase:** `ioypqogunwsoucgsnmla` (HGPG Core, shared with TC Concierge, CMA, brain-app, Team Tools)
- **PIN:** 2026
- **Stack:** Next.js + Supabase + Tailwind
- **Claude Code launch:** `cd ~/Documents/hgpg-transaction-manager && claude --dangerously-skip-permissions`

## Architecture

- **TC Concierge = intake; Transaction Manager = lifecycle.**
- Architecture doc has full ListedKit data model + ReZEN SC buyer & NC seller checklists.
- `/agent` route hosts the FUB AI Agent surface (see `projects/fub-ai-agent.md`)

## Active test transaction

- **4421 Magnolia Point Drive** (`15d83bd2-3c22-44d2-a427-18368713beb3`)
- Indian Land SC, buyer deal, under_contract
- Brian listed as buyer (Brian-as-client test pattern)
- Pro iMessage group: `loop_client_group_name="HGPG - 4421 Magnolia Point"` (pre-dates "- Pro"/"- Client" naming convention)
- Lamington Dr (`a7ac15e6`) is the CMA Engine test case only

## Recent ships

### Messaging (LoopMessage)

- **Twilio: FULLY DEPRECATED 2026-04**
- LoopMessage SaaS is the sole messaging path
- Group routing via `loop_client_group_id`
- Auto-completes iMessage group creation tasks on inbound webhook
- Client threads route through group when attached

### Milestone tracker with send-status badges

- `NotificationBadges` component in `MilestoneTrackerPanel.tsx`
- ✓/⚠ states for client_group + pro_group channels
- Reads `client_group_notified_at`, `pro_group_notified_at`, `notification_error`, `notification_skipped_reason`

### Milestone notification copy split

- `clientCopy` (warm/short/tracker link) + `proCopy` (terse, Don named on handoffs)
- Templates in `milestone_templates` DB table
- Closed-milestone client templates use `{{review_link}}` (Google for buyer, Zillow for seller)

### Multi-agent routing

- `transactions.coordinator_id`, `team_members.broker_oversight_enabled`, `zillow_review_url`
- `getDealNotificationRecipients` routes owner + coordinator + broker_oversight per deal
- Brian + Ashley have `broker_oversight=true`

### Post-close FUB handoff

- `lib/fubPostClose.ts` triggers on closed milestone
- Updates FUB fields, posts closing note, applies tags
- Action Plans path ripped out (commit `3d656dd`); cadence in FUB Automations 2.0 via tag triggers
- Idempotent via `post_close_fub_synced` audit row

### PWA push fan-out

- `lib/push.ts` with quiet hours 9pm-7am ET
- `dispatchPush({excludeUserId?, respectQuietHours?})`
- Actor exclusion so users don't buzz themselves

### Concierge auto-trigger

- Fires from `sendMilestoneNotification` at `clear_to_close` stage
- Session type `closing_checklist` (not `move_in/move_out`)

### Cron routing

- group-setup-nag + reminders use per-deal owner/coordinator lookups
- `TRACKER_NOTIFICATION_MODE=live`

### Don feedback batches

**5/1 batch (resolved):**
- Inspection/DD fee template surgery via `fix_inspection_and_dd_fee_templates_2026_05_01`
- SC has no DD fee (confirmed business rule)
- NC DD fee task offset contract+3

**5/6 Cluster A-D batch (7 items resolved):**
- DD fee placement
- Buyer inspection on seller side
- No vendor in Setup Inspection
- Seller concessions in summary email
- Termination fee not picked up
- Seller concessions $3000 not picked up
- Seller info link expired

### Other-side client email made optional (5/6)

- Seller/buyer wizards no longer require other-side email
- Format validation prevents typo'd addresses
- One-shot send only, no FUB promotion
- Other-side recipient gets transactional footer explaining why they got the email

### Seller Attorney Intake (5/6)

- PDF + deal-page button, firm-agnostic
- Sensitive fields routed directly to attorney

### transaction-pdfs bucket flipped private (PR #7, commit e45e3c8, merged ~2026-05-09)

- `lib/storage/signTransactionPdfUrl.ts` helper (7-day expiry, accepts path or legacy public URL, never throws)
- `lib/conciergePdf.tsx` persists storage path instead of public URL
- `app/concierge/[token]/page.tsx` signs server-side on every page load
- `app/api/rezen/push-document/route.ts` signs before email fanout
- Bucket flipped via Supabase MCP on 2026-05-09 (`schema_migrations` version `20260509173839`). Bucket `public = false` confirmed 2026-05-11.
- 2026-05-11: idempotent re-apply via MCP (`schema_migrations` version `20260511210146`) — no-op since already private, recorded for clarity.
- **Repo file pending:** `supabase/migrations/20260509_flip_transaction_pdfs_bucket_private.sql` staged in 2026-05-11 session; commit from Brian's Mac is the last loose thread.

### Meta Ads Dashboard (2026-05-19)

- `/meta-ads` route + `/api/meta-insights` API. Brian-only, gated server-side via `getSession()` + email check.
- Server-side Pipeboard JSON-RPC proxy. Token (`PIPEBOARD_API_TOKEN`) lives in Vercel env, never in browser. **Gotcha:** Vercel "Sensitive" env vars return as empty strings to runtime — re-add without the Sensitive toggle if `present:true, length:0` shows in `/api/debug-env`.
- Pipeboard endpoint `https://meta-ads.mcp.pipeboard.co/`, account `act_31445287`, 200/hour rate limit.
- 4 date tabs (7/14/30/90), KPI strip (Spend/Impressions/Clicks/Leads/Avg CPL), sortable table, CSV export, 5s refresh throttle.
- **Drill-down (2026-05-19 PM):** click any row to drill — campaigns → ad sets → ads. Breadcrumb at top for nav back up. Ad level is terminal. Unknown-status rows are dimmed and non-clickable (no current Pipeboard metadata to query against).
- **Hide unknown toggle** in tab row. Defaults ON. Count chip surfaces unknown count (e.g. "Hide unknown (12)").
- API query params: `level` (campaign|adset|ad), `parent_id`, `parent_kind` (campaign|adset). Backwards-compatible: no params = campaign-level.
- **60s response cache** per-lambda keyed by `${level}:${parentKind}:${parentId}:${days}` to absorb rapid drill-down clicks below Pipeboard's 200/hour limit.
- Tool name discovery (`tools/list`) cached per process (1hr TTL). Fail-soft on campaigns/adsets/ads metadata calls so status pills degrade to "unknown" but insights still render.
- Files: `app/meta-ads/page.tsx`, `app/meta-ads/MetaAdsDashboard.tsx`, `app/api/meta-insights/route.ts`, plus nav links in `app/layout.tsx` and `components/MobileNav.tsx`.
- **Bearer-token auth path (2026-05-20):** `/api/meta-insights` now also accepts `Authorization: Bearer $META_INSIGHTS_TOKEN`, short-circuiting the session+email gate. Lets the Ads project query insights directly without copy-paste. Session check preserved as fallback. Route added to middleware `PUBLIC_PATHS` so the route-level auth runs (middleware was returning 401 before the handler). Env var stored Sensitive=OFF.

#### Pipeboard MCP gotchas (learned the hard way 2026-05-19)
- **`get_campaigns` / `get_adsets` / `get_ads` ignore scoping args** like `campaign_id` and `adset_id`. They always return unscoped account-wide lists. Workaround: fetch wide, cache 5 min, look up by ID locally.
- **`get_insights` ignores the `filtering` array param.** Scope drill-downs via `object_id=<parent_id>` (mirrors Meta's native `/<id>/insights` URL-path scoping). The `filtering` array is sent as backup belt-and-suspenders.
- **`get_insights` ignores `date_preset` too** (confirmed 2026-05-21). Returns lifetime data for `last_7d`, `last_30d`, `last_90d` — all identical. Workaround: pass Meta's explicit `time_range: { since, until }` object instead. Pipeboard forwards this through to the Graph API correctly. Shipped in commit `96b256a` on `app/api/meta-insights/route.ts`.
- **Exact tool names matter.** Fuzzy `name.includes("ads")` matching can land on `get_ad_details` / `bulk_*` / `delete_*` etc. Pin to exact strings (`get_campaigns`, `get_adsets`, `get_ads`).
- Diagnostic endpoint `/api/debug-meta-ads` (in repo as of `36cdabb`, slated for removal) tests tool-call variants; useful template for other MCP integrations.


### deals -> transactions migration (DONE 2026-05-01)

- `hgpg-deals-tracker` decommissioned

## Coordinator

- **Don Broyles** is the sole TC, primary day-to-day TM user
- Submits feedback via built-in widget
- UUID `e3d56eb5-c5e9-46b7-9ff8-8b20e7da7f9a` (`DEFAULT_COORDINATOR_ID`)

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** ✅ 403 resolved (2026-05-09 status reconciliation confirmed working)

## Open items

- 🆕 **Don feedback 2026-05-13 — Ad-hoc transaction events (e.g. mid-deal signings)** — feedback ID `84a2761a-f5b9-40a7-ba3a-f7923e9f1b3f`, status=open in `feedback` table.
  - **Context:** On Lamington Dr (`a7ac15e6-bdfb-495a-8588-11e98692f905`, listing deal closing 2026-06-01), Don wants to schedule a seller deed signing for 2026-05-26 at 3 PM at Lando Law firm WITHOUT changing the 6/1 closing date. He asked "Wasn't sure just whether to put another task in?"
  - **Product gap:** TM auto-creates Google Calendar events for the 3 standard milestones (closing, DD deadline, walkthrough) but has no mechanism for ad-hoc transaction events like seller signings, intermediate inspections, lender meetings, attorney signings, etc. Don is asking for guidance because the right surface doesn't exist yet.
  - **Three plausible responses, ordered by build cost:**
    1. **Quick win (no build):** Tell Don to use a Task with a due date + time + location in the title. Works today. No calendar event though, no formal "this is a signing" semantic.
    2. **Calendar-only surface:** Add a "Schedule event" button on the deal page that opens a small form (event type, date/time, location, notes) and pushes to the HGPG Closings Google Calendar with reasonable defaults — title format `[Deal type] - [Event type] - [Address]`. Persist event ID in `transactions.calendar_event_ids` JSON. No new DB table.
    3. **Full Signing Event entity:** New `transaction_events` table with `event_type` enum (`seller_signing`, `buyer_signing`, `lender_meeting`, `inspection`, `attorney_signing`, `other`), `scheduled_at`, `location`, `attendees` (json), `notes`, `calendar_event_id`. Auto-syncs to shared calendar. Surfaces on deal page timeline. Recommended for medium-term; option 2 is a stepping stone.
  - **Recommendation:** Option 2 first (~2 hours), with the understanding that option 3 is the eventual destination once we see how often these ad-hoc events come up. Option 1 is too hand-wavy for a TC who is already feeling the pain.
  - **Don needs an answer in the meantime.** Whichever path is chosen, reply to him in the feedback queue. Until then, Option 1 ("Just add a task for now") is the right tactical answer for Lamington Dr specifically — won't block him from closing.
- **$395 fee toggle** — parked build spec, refs commit b9fa0deb. The underlying notes-append work for the 3-gate fee verification (contract distribution + mid-deal at under_contract+14 + settlement review) shipped 2026-05-05. Structural toggle still parked.
- **Migration file backfill** (`supabase/migrations/20260509_flip_transaction_pdfs_bucket_private.sql`) — repo hygiene only. DB ledger and bucket state both correct. Commit pending from Brian's Mac.

### Closed items

- ✅ **transaction-pdfs bucket flipped to private** (verified 2026-05-11). Code on main, bucket private, DB ledger correct since 2026-05-09. Only repo file commit remains.
- ✅ **NC office routing verified wired** (2026-05-11). Code at `app/api/rezen/create-transaction/route.ts:160` does `state === "NC" ? REZEN_OFFICE_ID_NC : REZEN_OFFICE_ID` with fallback, applied via `setOwnerInfo`. Both env vars confirmed set on Vercel Production. NC office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94` not hardcoded — pulled from env, which is the right pattern.
- ✅ Sherlock 403 resolved (2026-05-09)
- ✅ Lamington duplicate cleanup verified (single row in DB: `a7ac15e6-bdfb-495a-8588-11e98692f905`)
- ✅ Google Calendar OAuth integration shipped (calendar invites firing fine)

## Build patterns

- All work goes in `HGPG1/hgpg-transaction-manager` on `main`
- Brian pushes from his Mac via Claude Code with `--dangerously-skip-permissions`
- Web/sandbox sessions prepare files in `/mnt/user-data/outputs/` for Brian to download
- Verification gates: `tsc --noEmit` -> `npx eslint .` -> `next build`
- Commit author: `brian@homegrownpropertygroup.com` (NOT `brian@hgpg.com`)
- `npx eslint .` for lint gate (Next 16 removed `next lint`)

## DM Brian shortcut

- POST `/api/internal/dm-brian` on `closings.homegrownpropertygroup.com`
- Auth: Bearer token (in memory)
- Hardcoded recipient `+17046779191`
- 10/hour limit

