# Infrastructure

## Core stack

- **Standard:** Next.js App Router + Supabase + Tailwind
- **GitHub org:** HGPG1
- **Vercel team:** team_FietQPKCmnyioG2n0FdteQCV
- **DNS:** GoDaddy for homegrownpropertygroup.com

## Supabase projects

| Project | ID | Used for |
|---|---|---|
| Primary | ioypqogunwsoucgsnmla | TC Concierge, Transaction Manager, Deals Tracker |
| Listing Report Portal | wdheejgmrqzqxvgjvfee | reports.homegrownpropertygroup.com |
| Signature site (legacy) | fkxgdqfnowskflgbuxhm | signature.homegrownpropertygroup.com |

## Subdomains (all assigned in Vercel)

| Subdomain | Purpose |
|---|---|
| cma.homegrownpropertygroup.com | CMA Engine |
| closings.homegrownpropertygroup.com | Transaction Manager |
| tc.homegrownpropertygroup.com | TC Concierge |
| reports.homegrownpropertygroup.com | Listing Report Portal |
| deals.homegrownpropertygroup.com | Deals Tracker |
| buyersguide.homegrownpropertygroup.com | Buyers Guide (React + Vite) |
| signature.homegrownpropertygroup.com | Signature Marketing Collection |
| marketinganalyzer.homegrownpropertygroup.com | Property Marketing Analyzer (PIN: 1847) |
| search.homegrownpropertygroup.com | IDX Broker on WordPress (SiteGround) |

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** Currently returning 403 (likely API key scope issue)
- **NC office routing:** Use office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94` when `txn.state === 'NC'`

## MCP servers

- **FUB MCP:** `~/Documents/followupboss-mcp-server-neuhaus/index.js`

## iCloud workspace

`~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/`
- Syncs between home and work Macs
- Mirror/backup for this repo

## Meta / advertising

- **Ad Account:** 31445287
- **Pixel:** 4259881614223735

## Standard PINs

- CMA Engine: `2026`
- Transaction Manager: `2026`
- Deals Tracker: `2026`
- Property Marketing Analyzer: `1847`
- Signature admin: `hgpg2024admin` (at /admin)

## Security

- **GitHub PAT was exposed in chat:** `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` - rotate.
