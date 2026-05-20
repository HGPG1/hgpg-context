<!-- Last Updated: 2026-05-16 -->

# Infrastructure

> Secrets, project IDs, and credentials live in Claude memory and the Tech & Builds project instructions, not this file. This document covers structure only.

## Core stack

- **Standard:** Next.js App Router + Supabase + Tailwind on Vercel
- **GitHub org:** HGPG1
- **Vercel team:** HGPG (team ID in memory)
- **DNS:** GoDaddy for homegrownpropertygroup.com

## GitHub auth

### GitHub App (primary, since 2026-05-14)

- **App name:** HGPG Brain Commit
- **App ID:** 3712986
- **Installation ID:** 132322328 (HGPG1 org, installed on all repositories)
- **Permissions:** Contents read/write, Pull requests read/write (added 2026-05-16), Metadata read-only
- **Private key:** stored as Vercel env var `GITHUB_APP_PRIVATE_KEY` on brain-app project. Local backup at `~/.hgpg-secrets/github-app-private-key.pem` on iMac (chmod 600). Keep a copy in 1Password as well.
- **Auth helper:** `lib/octokit-app.ts` in brain-app repo. Mints fresh installation tokens (auto-rotate every hour).

### What this enables

- Claude sessions (any device, including mobile) can write to `hgpg-context` via `POST https://brain.homegrownpropertygroup.com/api/external/write`
- Claude sessions can read any file from any HGPG1 repo via `GET https://brain.homegrownpropertygroup.com/api/external/read?repo=...&path=...`
- Claude sessions can commit code directly to main on ANY HGPG1 repo via `POST https://brain.homegrownpropertygroup.com/api/external/commit`
- Claude sessions can open multi-file PRs against any HGPG1 repo via `POST https://brain.homegrownpropertygroup.com/api/external/pr` (auto-branch `claude/YYYY-MM-DD-<slug>`, appends to existing PR if branch already has one open)
- All four endpoints use the same bearer token (`BRAIN_WRITE_TOKEN` in Vercel env vars), stored in Claude memory and 1Password ("HGPG Brain Write Token")
- No PAT exists anywhere in the stack

### Workflow split (since 2026-05-16)

- **hgpg-context** (brain notes): write directly to main via `/api/external/write`. No review gate needed for brain files.
- **App repos** (hgpg-cma-tool, hgpg-transaction-manager, charlotte-sellers-guide-vercel, etc.): default to `/api/external/pr` for code changes. Use `/api/external/commit` (direct to main) only for: one-byte retriggers when Vercel misses a webhook, Brian explicitly requesting direct push, or emergency hotfix that can't wait.
- Multi-commit debug cycles land on a single review branch instead of polluting production with diagnostic commits.

### Local development

- gh CLI authenticated on Mac Mini and iMac (token in macOS Keychain) - used for normal git push from local clones, unchanged
- Brian still pushes from his Mac when editing locally and committing his own work - that's standard git, not affected by the App migration

### Rotation playbooks

- **Bearer token leaks** (`BRAIN_WRITE_TOKEN`): `vercel env rm BRAIN_WRITE_TOKEN production && vercel env add BRAIN_WRITE_TOKEN production` then redeploy. Update Claude memory with new value. ~60 seconds.
- **GitHub App private key leaks**: go to https://github.com/organizations/HGPG1/settings/installations/132322328, generate new private key, delete old one, update `GITHUB_APP_PRIVATE_KEY` env var on Vercel. App ID and Installation ID don't change.

### Historical note

- Prior to 2026-05-14, brain-app used a fine-grained PAT. All PATs were revoked 2026-05-13 after one was exposed in chat history. The GitHub App migration replaces PATs permanently.
- No PATs in remote URLs anywhere. Any PAT-embedded URL from older instructions is invalid.
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

Custom SMTP via Resend is wired into **two** Supabase projects (any new project that sends auth emails needs its own SMTP config — Supabase scopes SMTP per-project).

**HGPG Core** (`ioypqogunwsoucgsnmla`)
- Sender: `noreply@homegrownpropertygroup.com` (name: HGPG)
- Rate limit: 30/hr (was 2/hr on Supabase default)
- API key in 1Password as "Supabase HGPG Core SMTP"
- Affects: TM, CMA, TC Concierge, brain-app

**HGPG Listing Reports + MLS** (`wdheejgmrqzqxvgjvfee`) — added 2026-05-15
- Sender: `noreply@homegrownpropertygroup.com` (name: HGPG Team Dashboard)
- Rate limit: 30/hr (Supabase default once custom SMTP is enabled)
- API key shared with Core (re-used same Resend key)
- Affects: hgpg-team-dash magic-link auth, anything else later added to this Supabase project

**SMTP host config (both projects):**
```
Host:     smtp.resend.com
Port:     465
Username: resend
Password: re_... (Resend API key)
```

**Where it lives in Supabase dashboard:** Authentication → Notifications → Email → enable Custom SMTP. Rate limits are on Authentication → Rate Limits.

## Active subdomains

| Subdomain | Purpose | Repo |
|---|---|---|
| homegrownpropertygroup.com | Main site | homegrown-property-group-site |
| cma.homegrownpropertygroup.com | CMA Engine | hgpg-cma-tool |
| closings.homegrownpropertygroup.com | Transaction Manager (also hosts /agent FUB AI surface) | hgpg-transaction-manager |
| tc.homegrownpropertygroup.com | TC Concierge | tc-concierge |
| reports.homegrownpropertygroup.com | Listing Report Portal | hgpg-listing-report |
| brain.homegrownpropertygroup.com | Brain App (markdown editor for hgpg-context + commit API for all HGPG1 repos) | brain-app |
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
- `south-charlotte-report` + `south-charlotte-scraper` + `south-charlotte-market-report` - @southcharlottereport content brand pipeline (daily news via RSS+S3, monthly HGPG market video via repository_dispatch from market-report)

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

