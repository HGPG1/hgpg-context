<!-- Last Updated: 2026-05-07 (session 5) -->

# Infrastructure

> Secrets, project IDs, and credentials live in Claude memory and the Tech & Builds project instructions, not this file. This document covers structure only.

## Core stack

- **Standard:** Next.js App Router + Supabase + Tailwind on Vercel
- **GitHub org:** HGPG1
- **Vercel team:** HGPG (team ID in memory)
- **DNS:** GoDaddy for homegrownpropertygroup.com

## GitHub auth

- gh CLI authenticated on Mac Mini and iMac (token in macOS Keychain)
- Web/sandbox Claude sessions cannot push - they prepare files and Brian pushes from his Mac
- Brain App at `brain.homegrownpropertygroup.com` can write to `hgpg-context` directly via fine-grained PAT (single-user, magic link auth)
- No PATs in remote URLs. Any PAT-embedded URL from older instructions is invalid and revoked.
- Fix any leftover PAT remote: `git remote set-url origin https://github.com/HGPG1/REPO-NAME.git`

## Supabase projects (4 active)

| Display Name | Project ID | Purpose |
|---|---|---|
| HGPG Core | `ioypqogunwsoucgsnmla` | TC Concierge, Transaction Manager, CMA Engine, brain-app, Team Tools, FUB AI Agent |
| HGPG Listing Reports + MLS | `wdheejgmrqzqxvgjvfee` | Listing Report Portal + MLS Grid replication |
| HGPG FUB Integration | `ngdrliyjtqcwhhfrbxao` | Legacy FUB texting integration |
| HGPG Signature + Relocation | `fkxgdqfnowskflgbuxhm` | Signature site + NYC relocation lead capture |

**Removed 2026-05-04:** `nfwjgsanvmwmrginhklz` (Suna self-host, abandoned)

### HGPG Core — TM tables of note

- `fub_agent_lead_cooldowns` — added in session 4. Tracks per-lead cooldowns from rejected drafts and (session 5) inbound classifier outcomes. Columns: `id` (uuid), `fub_person_id` (bigint, matches existing `fub_agent_leads.fub_person_id` type), `cooldown_until` (timestamptz), `reason` (enum), `created_at`, `created_by_draft_id` (uuid FK to `fub_agent_message_drafts.id`). Indexed on `(fub_person_id, cooldown_until)`. Reason enum (session 5): `voice_off | wrong_lead_type | wrong_signal_read | low_quality | other | hostile_reply | not_now_reply`. `draftGenerator` skips leads with active cooldown.
- `fub_agent_lead_optouts` — session 1 table; session 5 wired the inbound classifier to write here on `opt_out` classification. Schema: `fub_person_id` (bigint), `optout_source` (enum incl. `inbound_email`, `llm`), `optout_message` (truncated body), `optout_confidence` (numeric), `detected_channel` (`imessage|email|sms|phone|manual`), `notes`, `created_at`. The inbound classifier also flips `fub_agent_leads.is_eligible = false` with `eligibility_reason = 'inbound_optout'` on the same call. `draftGenerator` skips any lead with an optout row (belt+suspenders alongside `is_eligible`).
- `fub_agent_message_drafts` — session 4 added columns: `fub_pushed_at` (idempotency marker for pusher), `reject_reason` (enum), `reject_notes` (free-form, optional).

## Follow Up Boss (FUB)

### FUB AI Agent custom fields and tags

Custom fields (created via FUB API in session 4, IDs 157-159):
- `customAgentDraftEmailBody` (id 157, long text, admin-only) — written by `lib/agent/fubPusher.ts` on email-channel approval
- `customAgentDraftEmailSubject` (id 158, short text, admin-only) — written by pusher on email-channel approval
- `customAgentDraftIMessageBody` (id 159, long text, admin-only) — written by pusher on imessage-channel approval

Tags (implicit-on-first-apply since FUB tag list endpoint is not accessible to the API key):
- `agent_draft_email_ready` — applied by pusher on email-channel approval
- `agent_draft_imessage_ready` — applied by pusher on imessage-channel approval

Single source of truth for these names: `lib/agent/fubAgentConstants.ts` in the TM repo.

### FUB-related env vars

- `FUB_AGENT_INBOUND_SECRET` — required by `app/api/agent/inbound/route.ts` (TM). Route returns 503 until set. Session 5 wired the classifier-side effects (optout + cooldown writes), but Brian still needs to add this var in Vercel before pointing FUB at the inbound webhook.

## TM cron jobs

Registered in `hgpg-transaction-manager/vercel.json`. All UTC schedules.

| Path | Schedule (UTC) | Notes |
|---|---|---|
| `/api/cron/reminders` | `0 13 * * *` | Daily deadline reminders (8am EST) |
| `/api/cron/milestone-automation` | `15 13 * * *` | Milestone automation pass |
| `/api/cron/concierge-reminders` | `0 14 * * *` | TC Concierge reminders |
| `/api/cron/sync-emails` | `*/5 * * * *` | Gmail->transaction email matching |
| `/api/cron/group-setup-nag` | `5 13 * * *` | iMessage group nag |
| `/api/cron/feedback-digest` | `0 22 * * *` | Daily feedback digest |
| `/api/agent/cron` | `0 7 * * *` | FUB AI Agent nightly sync + score (no-op while `agent_enabled=false`) |
| `/api/cron/agent-daily-flush` | `1 5 * * *` | FUB AI Agent — flush approved-but-unpushed drafts at start of new ET day. UTC `1 5 * * *` = 12:01 AM EST or 1:01 AM EDT (DST drift acceptable; only needs to fire after the ET day rolls over). No-op while `agent_enabled=false`. Added session 5. |

## Resend SMTP

- Custom SMTP wired into HGPG Core Supabase
- Sender: `noreply@homegrownpropertygroup.com` (name: HGPG)
- Rate limit: 30/hr (was 2/hr on Supabase default)
- API key in 1Password as "Supabase HGPG Core SMTP"
- Affects ALL apps on HGPG Core: TM, CMA, TC Concierge, brain-app, Team Tools

## Active subdomains

| Subdomain | Purpose | Repo |
|---|---|---|
| homegrownpropertygroup.com | Main site | homegrown-property-group-site |
| cma.homegrownpropertygroup.com | CMA Engine | hgpg-cma-tool |
| closings.homegrownpropertygroup.com | Transaction Manager (also hosts /agent FUB AI surface) | hgpg-transaction-manager |
| tc.homegrownpropertygroup.com | TC Concierge | tc-concierge |
| reports.homegrownpropertygroup.com | Listing Report Portal | hgpg-listing-report |
| brain.homegrownpropertygroup.com | Brain App (markdown editor for hgpg-context) | brain-app |
| tools.homegrownpropertygroup.com | HGPG Team Tools | hgpg-team-tools2 |
| newconstruction.homegrownpropertygroup.com | Charlotte New Construction | charlotte-new-construction-nextjs |
| marketinganalyzer.homegrownpropertygroup.com | Property Marketing Analyzer | hgpg-property-analyzer |
| calls.homegrownpropertygroup.com | Call script tool | hgpg-call-script |
| buyersguide.homegrownpropertygroup.com | Buyers Guide (React + Vite) | charlotte-buyers-guide |
| sellersguide.homegrownpropertygroup.com | Sellers Guide (Meta Pixel + CAPI live) | charlotte-sellers-guide-vercel |
| signature.homegrownpropertygroup.com | Signature Marketing Collection | signature-marketing-collection1 |
| relocate.homegrownpropertygroup.com | NYC-to-Charlotte relocation landing | nyc-to-charlotte-relocation |
| portfolio.homegrownpropertygroup.com | Portfolio site | hgpg-portfolio |
| search.homegrownpropertygroup.com | IDX Broker on WordPress (SiteGround) | n/a |

## Active build repos (not subdomain-mapped above)

- `hgpg-context` - this brain repo
- `hgpg-imessage-bridge` - backs the closings.../api/internal/dm-brian relay
- `hgpg-home-calculator` - home calculator tool
- `closing-concierge` - smart home manufacturer link database (decommission pending)
- `south-charlotte-report` + `south-charlotte-scraper` - @southcharlottereport content brand pipeline

## Archived

- `hgpg-deals-tracker` - decommissioned 2026-05-01, superseded by hgpg-transaction-manager
- `hgpg-transaction-monitor` - superseded by hgpg-transaction-manager

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** Currently returning 403 (likely API key scope issue)
- **NC office routing:** Swap to NC-specific office ID when `txn.state === 'NC'` (ID in memory)

## MCP servers

- **FUB MCP:** `~/Documents/followupboss-mcp-server-neuhaus/index.js` - Claude Desktop only

## iCloud workspace

`~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/`
- Syncs between home and work Macs
- Mirror/backup for this repo

## PINs and admin credentials

All in memory. PIN 2026 standard across CMA Engine, Transaction Manager. PIN 1847 for Property Marketing Analyzer.
