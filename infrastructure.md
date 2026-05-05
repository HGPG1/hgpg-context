# Infrastructure

> Secrets, project IDs, and credentials live in Claude memory and the Tech & Builds project instructions, not this file. This document covers structure only.

## Core stack

- **Standard:** Next.js App Router + Supabase + Tailwind on Vercel
- **GitHub org:** HGPG1
- **Vercel team:** HGPG (team ID in memory)
- **DNS:** GoDaddy for homegrownpropertygroup.com

## GitHub auth

- gh CLI authenticated on Mac Mini and iMac (token in macOS Keychain)
- Web/sandbox Claude sessions cannot push — they prepare files and Brian pushes from his Mac
- No PATs in remote URLs. Any PAT-embedded URL from older instructions is invalid and revoked.
- Fix any leftover PAT remote: `git remote set-url origin https://github.com/HGPG1/REPO-NAME.git`

## Supabase projects (5 active)

- **Primary** (`tc-concierge` named) — shared by TC Concierge, Transaction Manager, CMA Engine
- **Listing Report Portal** — dedicated
- **Closing Concierge** — dedicated
- **FUB Texting Integration** — dedicated
- **Signature site** — legacy, dedicated

Project IDs in memory and Tech & Builds project instructions.

## Active subdomains

| Subdomain | Purpose | Repo |
|---|---|---|
| homegrownpropertygroup.com | Main site | homegrown-property-group-site |
| cma.homegrownpropertygroup.com | CMA Engine | hgpg-cma-tool |
| closings.homegrownpropertygroup.com | Transaction Manager | hgpg-transaction-manager |
| tc.homegrownpropertygroup.com | TC Concierge | tc-concierge |
| reports.homegrownpropertygroup.com | Listing Report Portal | hgpg-listing-report |
| marketinganalyzer.homegrownpropertygroup.com | Property Marketing Analyzer | hgpg-property-analyzer |
| calls.homegrownpropertygroup.com | Call script tool | hgpg-call-script |
| buyersguide.homegrownpropertygroup.com | Buyers Guide (React + Vite) | charlotte-buyers-guide |
| sellersguide.homegrownpropertygroup.com | Sellers Guide | charlotte-sellers-guide-vercel |
| signature.homegrownpropertygroup.com | Signature Marketing Collection | signature-marketing-collection1 |
| relocate.homegrownpropertygroup.com | NYC-to-Charlotte relocation landing | nyc-to-charlotte-relocation |
| portfolio.homegrownpropertygroup.com | Portfolio site | hgpg-portfolio |
| search.homegrownpropertygroup.com | IDX Broker on WordPress (SiteGround) | n/a |

## Active build repos (not subdomain-mapped above)

- `hgpg-context` — this brain repo
- `hgpg-imessage-bridge` — backs the closings.../api/internal/dm-brian relay
- `hgpg-home-calculator` — home calculator tool
- `closing-concierge` — smart home manufacturer link database
- `charlotte-new-construction-nextjs` — new construction site (active build)
- `south-charlotte-report` + `south-charlotte-scraper` — @southcharlottereport content brand pipeline

## Archived

- `hgpg-deals-tracker` — replaced/abandoned
- `hgpg-transaction-monitor` — superseded by hgpg-transaction-manager

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** Currently returning 403 (likely API key scope issue)
- **NC office routing:** Swap to NC-specific office ID when `txn.state === 'NC'` (ID in memory)

## MCP servers

- **FUB MCP:** `~/Documents/followupboss-mcp-server-neuhaus/index.js` — Claude Desktop only

## iCloud workspace

`~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/`
- Syncs between home and work Macs
- Mirror/backup for this repo

## PINs and admin credentials

All in memory. PIN 2026 standard across CMA Engine, Transaction Manager. PIN 1847 for Property Marketing Analyzer.
