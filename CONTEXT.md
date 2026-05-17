# HGPG Context (Brain)

Last updated: 2026-05-17

## Who

**Brian McCarron** - Team lead and owner-agent, Home Grown Property Group (HGPG). HGPG is a team operating under Real Broker LLC (the brokerage). Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for the full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## How to update the brain

The brain has two write paths. Claude sessions use the write API. Brian uses the web editor or `brain` shell function.

**Claude sessions:** `POST https://brain.homegrownpropertygroup.com/api/external/write` with bearer token. Body `{path, content, message}`. Creates or overwrites only; no delete. Always GET the raw github URL or `/api/files/<path>` first to read current contents, then POST the spliced file back. Health check: `GET` the same endpoint.

**For app code (any HGPG1 repo):** `POST /api/external/commit` with the same bearer token. Body `{repo, path, content, message}`. Available since 2026-05-14. Same safety rules.

**Brian:** `brain.homegrownpropertygroup.com` magic-link UI (any device) or `~/Documents/hgpg-context` with the `brain` shell function.

**Reading the brain programmatically:** `GET /api/files` for the full tree, `GET /api/files/<path>` for a single file. Both require the same bearer token (since 2026-05-15). For unauthenticated reads, use `raw.githubusercontent.com/HGPG1/hgpg-context/main/<path>` (cache may lag by minutes - use the API for fresh state).

## What's active right now

- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, multi-agent routing live, post-close FUB handoff wired. See `projects/transaction-manager.md`.
- **TC Concierge embedded in TM** at `/intake` - Don runs real deals through it. The standalone `tc.homegrownpropertygroup.com` is 503 (decommissioned). See `projects/tc-concierge.md`.
- **CMA Engine** (cma.homegrownpropertygroup.com) - active build on Matrix (Canopy MLS, NOT Paragon). GLA fix shipped, $60/sqft charlotte-metro contributory rate. See `projects/cma-engine.md`.
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com) - Selling Score v2 live, Meta Pixel + CAPI live, Meta ads launch pending. Lead data in HGPG Signature + Relocation Supabase, NOT HGPG Core. See `projects/sellers-guide.md`.
- **Charlotte New Construction** (newconstruction.homegrownpropertygroup.com) - shipped, 4 FUB-routed lead capture forms, Meta Pixel + CAPI live. Active sub-projects: incentives funnel, NC quiz scoring, builder submit footer link, scout admin cleanup, phone capture. See `projects/new-construction.md` and the `projects/new-construction-*.md` set.
- **Brain App** (brain.homegrownpropertygroup.com) - shipped, MVP + Phase 1.5 + write/commit APIs + GitHub App migration + /api/files auth gate. See `projects/brain-app.md`.
- **FUB AI Agent** - active build, see `projects/fub-ai-agent.md`.
- **Team Dashboard / Team Tools / Team Photo Sync** - see `projects/team-dashboard.md`, `projects/team-tools.md`, `projects/team-photo-sync.md`.
- **PropStream Caller** - see `projects/propstream-caller.md`.
- **Geo-farming postcards** - June 2026 drop ready (Bent Creek, Bridgemill, Queensbridge). See latest SESSION-HANDOFF.md entry.
- **Claude skills** - listing-wizard, client-journey-wizard, manus-prompt-architect, brians-prompt-genie, plus five from May (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer). See `projects/claude-skills.md`.

## Recently completed

- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite
- Sellers guide rebrand to brand colors
- Brain App write API (Phase 1.6) and commit API (2026-05-14) shipped
- GitHub App migration replacing all PATs (2026-05-14)
- /api/files auth gating (2026-05-15)
- Resend SMTP wired into HGPG Core and HGPG Listing Reports + MLS Supabase projects
- NeverBounce email validation on incentives form
- TM 395 fee toggle

## Standing infra rules

- Twilio/A2P fully deprecated (removed April 2026). iMessage via LoopMessage is the only TM messaging path.
- FUB Action Plans deprecated. Use FUB Automations 2.0.
- HGPG is a team. Real Broker LLC is the brokerage. Never call HGPG "the brokerage" in copy or code.
- All times in ET with UTC equivalent in parentheses where ambiguous.
- Refer to Supabase projects by friendly name: HGPG Core, HGPG Listing Reports + MLS, HGPG Signature + Relocation, HGPG FUB Integration.
- Vercel commit author always brian@homegrownpropertygroup.com.

## Known blockers / pending

See `SESSION-HANDOFF.md` for the current scratchpad.

- Sherlock 403 on Transaction Manager (likely API key scope) - status unclear, verify on next TM work
- .net Google Workspace migration to .com (do not proactively remind)
- MLS Grid API token from Canopy still pending (contact Bridgett Bouvier, data@canopyrealtors.com)
- Sellers Guide Meta ads launch
- TM remaining: addendum body block injection, per-party messaging capability tracking, calendar gate logic backfill
- Agent TM onboarding Phase 3 (Ashley, Taylor, Brenda)

## Decommissioned (do not reference as live)

- `transactions.homegrownpropertygroup.com` (hgpg-transaction-monitor) - archived, table renamed `_archived_transaction_emails_2026_04_29`
- `concierge.homegrownpropertygroup.com` standalone - 503, replaced by TM-embedded `/intake`
- `tc.homegrownpropertygroup.com` standalone - decommissioned, embedded in TM
- Manus, Make.com, Railway, Suna - all retired
- Twilio/A2P stack - fully deprecated April 2026

## Where to look next

| Topic | File |
|---|---|
| Brand colors, fonts, voice | `brand.md` |
| Team, FUB IDs, contractors | `team.md` |
| GitHub/Vercel/Supabase/DNS | `infrastructure.md` |
| SC compliance, Lando Law | `compliance.md` |
| Meta Ads, lead re-engagement | `marketing.md` |
| Session protocol, formatting, iMessage | `operations.md` |
| Specific project state | `projects/<name>.md` |
| Last session pickup | `SESSION-HANDOFF.md` |
