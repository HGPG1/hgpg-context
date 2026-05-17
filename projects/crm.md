<!-- Last Updated: 2026-05-17 -->

# CRM & Lead Management

Active FUB campaign state, lead pipeline experiments, tagging strategy, and Automations 2.0 builds for HGPG.

For the HGPG - CRM & Leads Claude project instructions, see the project sidebar in Claude. This file is the brain's record of what's actually running.

## Status

🟡 Stub. Created 2026-05-17 to give CRM/FUB work a canonical place to land. Most live FUB state currently lives in session handoffs and memory; consolidate here as work happens.

## Lead pipeline (high-level)

- ~13,000 total leads in FUB
- Origin mix: Ylopo (legacy, 10,738 extracted), Showcase IDX (cancelled), IDX Broker (current, Realty Candy Account 69061), expired/canceled/FSBO seller lead feeds, organic site capture (New Construction, Sellers Guide, Charlotte buyer guide)
- Top 25 active buyer leads tagged `Top25ActiveBuyers`, assigned to Taylor (FUB user ID 21)
- Pond 16 (idxRE engagement pond) recently segmented into Seller / Buyer / Unknown tags

## Active campaigns

(To fill in as campaigns are running. Each campaign should have: name, audience, trigger, Automations 2.0 ID or surface, KPIs being watched, start date.)

## Recent FUB work

(Append as sessions ship FUB-related changes. Date stamp each entry. Goal: this section tells the next session what just changed in FUB so they don't re-do or contradict.)

## Standing FUB conventions

- Automations 2.0 only. Action Plans are deprecated. Never propose `/people/{id}/actionPlans` or Action Plan creation.
- FUB MCP hosted at fub.homegrownpropertygroup.com (works in any Claude session). The old Claude-Desktop-only local install is deprecated.
- `/calls` endpoint is unreliable - use `/notes` for call logging.
- `/people` PUT accepts `assignedUserId` (integer) and `mergeTags` as query param, not body.
- Smart list creation is NOT supported via API - manual UI work in FUB.
- Mass Actions do NOT fire automations - re-running requires re-firing the trigger or cloning with new conditions.
- DNC tag: `Y_DNC_REGISTRY_TRUE`
- ISA pseudonym in FUB: "Amanda Morgan" (not a real person)

## Connected systems

- **Transaction Manager** (closings.homegrownpropertygroup.com): post-close handoff to FUB via `lib/fubPostClose.ts`. Updates lead on `closed` milestone, applies tags. Post-close cadence runs in Automations 2.0.
- **Charlotte New Construction** (newconstruction.homegrownpropertygroup.com): 4 FUB-routed lead capture forms. Email template IDs in projects/charlotte-new-construction.md.
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com): Selling Score v2 captures seller leads. Lead data lands in HGPG Signature + Relocation Supabase (fkxgdqfnowskflgbuxhm), NOT HGPG Core.
- **IDX Broker** (Realty Candy account 69061): webhook integration to FUB. Lead routing configured to Brian, FUB Embedded App live in contact profiles.
- **South Charlotte Report** (south-charlotte-report.md): daily content pipeline, not FUB-integrated directly.

## Tooling

- **fub-texting-integration** repo (HGPG1) + HGPG FUB Integration Supabase (ngdrliyjtqcwhhfrbxao). Intended foundation for the planned `leads.homegrownpropertygroup.com` rebuild.
- **FUB AI Agent** (projects/fub-ai-agent.md): in-TM agent at closings.homegrownpropertygroup.com/agent. Sessions 4-5 in progress.
- **PropStream Caller** (projects/propstream-caller.md): cold-calling pipeline, parked/abandoned.

## Compliance

- NC: NCREC Rule 58A .0104 prohibits cold-calling new prospects without a license. Tighter than SC.
- Twilio/A2P deprecated April 2026. No outbound SMS path to FUB leads currently exists. Flag if a use case emerges - don't propose reviving Twilio.

## Phone numbers

- Brian primary: 704-677-9191 (default for content/outreach)
- FUB line: 803-902-4701 (only when explicitly requested)
