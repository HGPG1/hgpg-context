# Infrastructure

> Secrets, project IDs, and credentials live in Claude memory, not this file. This document covers structure only.

## Core stack

- **Standard:** Next.js App Router + Supabase + Tailwind
- **GitHub org:** HGPG1
- **Vercel team:** HGPG (team ID in memory)
- **DNS:** GoDaddy for homegrownpropertygroup.com

## Supabase projects (3 total)

- **Primary** - shared by TC Concierge, Transaction Manager, Deals Tracker
- **Listing Report Portal** - dedicated
- **Signature site** - legacy, dedicated

Project IDs in memory.

## Subdomains

| Subdomain | Purpose |
|---|---|
| cma.homegrownpropertygroup.com | CMA Engine |
| closings.homegrownpropertygroup.com | Transaction Manager |
| tc.homegrownpropertygroup.com | TC Concierge |
| reports.homegrownpropertygroup.com | Listing Report Portal |
| deals.homegrownpropertygroup.com | Deals Tracker |
| buyersguide.homegrownpropertygroup.com | Buyers Guide (React + Vite) |
| signature.homegrownpropertygroup.com | Signature Marketing Collection |
| marketinganalyzer.homegrownpropertygroup.com | Property Marketing Analyzer |
| search.homegrownpropertygroup.com | IDX Broker on WordPress (SiteGround) |

## ReZEN APIs

- **Arrakis:** `arrakis.therealbrokerage.com/api/v1` - uses `X-API-KEY` header
- **Sherlock:** Currently returning 403 (likely API key scope issue)
- **NC office routing:** Swap to NC-specific office ID when `txn.state === 'NC'` (ID in memory)

## MCP servers

- **FUB MCP:** `~/Documents/followupboss-mcp-server-neuhaus/index.js`

## iCloud workspace

`~/Library/Mobile Documents/com~apple~CloudDocs/HGPG-Cowork/`
- Syncs between home and work Macs
- Mirror/backup for this repo

## PINs and admin credentials

All in memory. Standardized across most projects.
