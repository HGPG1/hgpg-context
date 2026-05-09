<!-- Last Updated: 2026-05-09 -->

# Session Handoff

## Last session: 2026-05-09 (PM) — Status reconciliation + brain refresh 🟢

### What this session was

A line-by-line walkthrough of every active and on-the-horizon project to reconcile what's actually shipped against what the brain said. Brian fed answers verbally; Claude verified key items via Supabase + Vercel MCP and wrote the diffs back to the brain.

### Confirmed state changes (close-outs)

- **TM Sherlock 403** → **resolved**. Off the open-items list.
- **TM Lamington duplicate cleanup** → **resolved**. Verified single row in DB (`a7ac15e6-bdfb-495a-8588-11e98692f905`, 7807 Lamington Drive, under_contract).
- **Google Calendar OAuth integration for TM** → **shipped**. Calendar invites firing fine. Off the on-the-horizon list.
- **MLS Grid auto-pull comps for CMA Engine** → **deep into integration / mature**. Bridgett's Bearer token landed; primary data source for CMA comps via `searchComps()`. Off the on-the-horizon list.
- **FUB API key swap on sellers guide** → **done**. Test leads landing fine.
- **Suna project** → **fully torn down**. Already removed from active Supabase list.
- **Closing Concierge subdomain (`concierge.homegrownpropertygroup.com`)** → **confirmed torn down**. Verified via Vercel MCP — no `closing-concierge` project in team. DNS likely still has CNAME pointing nowhere (cleanup task for GoDaddy).
- **Meta Pixel / CAPI playbook rollout scope reduced**. Marketing Analyzer + TM = NOT NEEDED. Signature = deferred. Only Signature remains parked.

### Confirmed in-flight or just-shipped

- **TM transaction-pdfs bucket flip → private + signed URLs** kicked off as Claude Code task during this session. Spec covers `lib/conciergePdf.tsx:38/404/416`, `app/api/send-task-email/route.ts:495`, `app/api/rezen/push-document/route.ts:108`. Goal is private bucket with on-demand 7-day signed URLs at every read site, no lifecycle pruning.
- **NewCon site is now serving live Meta ad traffic.** Pixel + CAPI plumbing is load-bearing for ad spend.
- **Buyers Guide rebuild well under way.** Active.
- **NewCon is at full primetime / serving as the showcase build.**

### FUB AI Agent — gate status

Viktor's Automation 2.0 setup is **complete**. **Smoke test has NOT been run.** Brian sitting at the gate.

The 4-checkpoint smoke test sequence is unchanged from prior session. Still:
1. Manual test through the Automation
2. Confirm email arrives plaintext with merge tags resolved
3. Run `scripts/session-5-smoke-test.sql` Section A→B→C
4. Push one real warm draft (Gerard, draft id 7, score 37)

Then flip `agent_enabled = true`, ramp 10 → 25/day over week 1.

`agent_enabled` still false. Cron healthy and gated. Queue still has 4 warm drafts ready (Gerard, Seanna, Jay, Anthony Scott).

### Pinned strategic items (no work, just flagged)

- **Consolidation opportunity: Listing Report Portal + Team Dashboard + CMA Engine.** All three lean on the same MLS Grid replication, all care about team-involved listings, all behind PIN/magic-link auth. Real argument for one unified "HGPG MLS Workspace" — counterargument is seller-facing polish bar differs from internal tooling. Park for strategic review session.
- **NewCon ads running** — Pixel/CAPI spot-checks every couple weeks in Meta Events Manager warranted. Silent break = wasted ad budget.

### Rainy day items added

- **Vercel project cleanup audit.** 8 candidates appear dormant or superseded:
  - `hgpg-transaction-monitor` (archived 4-29 per memory but Vercel project still present)
  - `fub-texting-integration` (legacy)
  - `hgpg-listing-admin`
  - `hgpg-sites-dashboard`
  - `hgpg-blog-dash`
  - `hgpg-blog-automation-dashboard`
  - `hgpg-calc`
  - `hgpg-lead-caller`
  - `charlotte-neighborhood-guides`
  
  Plus DNS cleanup in GoDaddy — remove `concierge` CNAME.

### What did NOT change

- TM NC office routing (NC office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94`) — Brian unsure, never confirmed wired into ReZEN builder. Still open.
- TM $395 fee toggle (build spec parked, refs `projects/tm-395-fee-toggle.md`).
- South Charlotte Report — clean, running, no changes.
- TC Concierge — not directly used anymore, but TM depends on its data path; left as-is.
- NC Scout / Incentive Concierge — Brian flagged dashboard clutter from failed manual syncs (small follow-up: hide button + auto-hide after N failures). Not yet built.

### Pickup notes for next session

- Watch for the transaction-pdfs bucket flip PR to land. After it merges and Vercel deploys, smoke-test: 1) magic-link concierge "Download Summary PDF", 2) TM per-task "Send Email" on an `attach_source_pdfs` template, 3) direct hit on a public bucket URL should 400/404.
- Then circle back to FUB AI Agent smoke test gate. Desktop session preferred (multi-window watching).
- NC Scout dashboard hide-failed-syncs task — small (2-3 hr), nice fire-and-forget.
- Buyers Guide rebuild push to finish line.

---

## Prior session: 2026-05-09 (AM) — FUB Agent session 6 stopping point 🟡

### What got shipped (since brain commits b90d85a / ffcefd9)
- **FUB email template id 1156 created via API** (`Agent Draft Outreach (System)`). Subject `[-customAgentDraftEmailSubject-]`, body `[-customAgentDraftEmailBody-]`. Round-trip verified — both merge tags came back byte-identical via GET `/v1/templates/1156`. Created `2026-05-09T09:51:30Z` UTC / `05:51:30-04:00` ET.
- **`scripts/fub-email-shell.md`** updated with a "Template created" footer block (id, ISO timestamps, verification status, next-step instruction for Viktor).
- **Repo-root `CLAUDE.md`** created with a session-end checklist: push code, push brain via `POST https://brain.homegrownpropertygroup.com/api/external/write`, verify 2xx, no `_brain-pending/` as default staging.
- **`BRAIN_WRITE_TOKEN` moved to env var.** Value lives in `.env.local` (gitignored); `.env.local.example` shipped with the var name only. `.gitignore` updated with `!.env*.example` (the existing `.env*` rule was over-broad) and `_brain-pending/` (so fallback staging never lands in git).
- **Pre-flight checks both green** before Viktor's done:
  - GET `/v1/customFields` confirmed exact-name match for `customAgentDraftEmailBody` (157) and `customAgentDraftEmailSubject` (158). Capitalization, prefix, casing all align with `lib/agent/fubAgentConstants.ts`.
  - Supabase queue: 4 drafts in `pending_review`. Top 3 by combined_score: Gerard (draft 7, 37, viewed_listing_recent.email), Seanna (draft 8, 37, same), Jay (draft 9, 35, seller_report_engaged.seller). Anthony Scott is the 4th. Real warm-tier email drafts ready to push when smoke-test time comes.
- **`fub_cleanup/` decommissioned.** Stale FUB API key (`...wz1BQa`) sat in plaintext in an iCloud-synced dir for ~2 weeks. Verified against FUB Admin: Brian had already revoked it in earlier cleanup, so it was inert by the time it was caught. `fub_cleanup/.env` replaced with REVOKED marker; `fub_cleanup/README.md` got a status banner. Whole dir gitignored. Production FUB key (`...5j9xeR` in `.env.local` / `.env.vercel.local`) is separate and untouched.

### What did NOT ship (in AM session)
- `agent_enabled` still **false**. No outbound. Cron healthy and gated.
- Smoke-test SQL not run.
- (PM session update: Viktor's Automation now wired and complete; smoke-test gate is the only remaining step before flip.)

---

## Locked queue

1. ⏳ Sellers guide Pixel QA — close out before treating as fully done
2. 🔄 Team Dashboard Tab 2 (Market Intelligence). Was next; FUB Agent jumped ahead this session.
3. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (~4-5 hr)
4. 🔍 Off-market / Expireds Finder — (~5-6 hr, replaces external scraper)
5. 🤖 FUB AI Agent: smoke test + go-live (next session)
6. 🪣 TM transaction-pdfs bucket private+signed (in flight as of 2026-05-09 PM)

### Smaller open items
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)
- NC Scout dashboard: hide failed manual syncs (2-3 hr)
- TM NC office routing verification (need to confirm conditional swap is wired into ReZEN builder)

### Parked
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard now covers the team-facing need)
- Listing Report Portal + Team Dashboard + CMA Engine consolidation review (strategic, no work yet)
- Vercel project cleanup audit (rainy day, 8 candidates listed above)
- TM $395 fee toggle
