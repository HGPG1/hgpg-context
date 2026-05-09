<!-- Last Updated: 2026-05-09 -->

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

- **NC office routing** — swap to NC office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94` when `txn.state === 'NC'`. Brian unsure if conditional swap is wired into ReZEN builder. Needs verification.
- **transaction-pdfs bucket flip → private + signed URLs** — Claude Code task kicked off 2026-05-09. Spec covers `lib/conciergePdf.tsx` (3 lines: 38, 404, 416), `app/api/send-task-email/route.ts:495`, `app/api/rezen/push-document/route.ts:108`. Goal: private bucket + on-demand 7-day signed URLs at every read site, no lifecycle pruning. **Bucket usage is mixed-purpose long-term storage** (concierge magic-link PDFs read weeks/months later, source-PDF re-attach on `attach_source_pdfs` task templates fires days/weeks after intake), so blanket lifecycle policy ruled out.
- **$395 fee toggle** — parked build spec, refs commit b9fa0deb. The underlying notes-append work for the 3-gate fee verification (contract distribution + mid-deal at under_contract+14 + settlement review) shipped 2026-05-05. Structural toggle still parked.

### Closed items (2026-05-09 reconciliation)

- ✅ Sherlock 403 resolved
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
