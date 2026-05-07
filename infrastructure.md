<!-- Last Updated: 2026-05-07 -->

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

- `fub_agent_lead_cooldowns` — added in session 4. Tracks per-lead cooldowns from rejected drafts. Columns: `id` (uuid), `fub_person_id` (bigint, matches existing `fub_agent_leads.fub_person_id` type), `cooldown_until` (timestamptz), `reason` (enum: `voice_off | wrong_lead_type | wrong_signal_read | low_quality | other`), `created_at`, `created_by_draft_id` (uuid FK to `fub_agent_message_drafts.id`). Indexed on `(fub_person_id, cooldown_until)`. `draftGenerator` skips leads with active cooldown.
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

- `FUB_AGENT_INBOUND_SECRET` — required by `app/api/agent/inbound/route.ts` (TM). Currently UNSET — route returns 503 until set. Will be wired in session 5 when inbound classifier connects to optout + cooldown tables.

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
