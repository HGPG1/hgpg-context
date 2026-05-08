<!-- Last Updated: 2026-05-08 -->

# PropStream Caller

- **Status:** ⚫ Parked / mostly abandoned, status unclear
- **URL:** leads.homegrownpropertygroup.com
- **Repo:** unknown — possibly still on Manus, possibly migrated to HGPG1 GitHub
- **Distinct from:** the in-TM FUB AI Agent (that one is `projects/fub-ai-agent.md`, lives at `closings.homegrownpropertygroup.com/agent`, currently at sessions 4+5)
- **Distinct from:** the Manus-era FUB AI Agent + Texting Integration suite (decommissioned May 2026, spec preserved at `archive/fub-agent-handoff.md`)

## What it is (or was)

A cold-calling pipeline driven by leads imported from PropStream. Used to identify and dial property owners outside HGPG's existing CRM pool — the kind of "off-market / motivated seller" outreach that PropStream specializes in.

Workflow at peak: load PropStream lead lists into the system, work through them via outbound calling, capture outcomes.

## Current state

- Lead import phase happened (PropStream → wherever the system stores leads)
- Calling phase began but was not completed
- Project is mostly abandoned in current state
- No active development, no active calling
- Live at `leads.homegrownpropertygroup.com` (so something IS deployed)

## Open questions to resolve when this is reactivated

These are the things we don't currently know but would need to know:

- **Where's the code?** Manus may still own it, OR it may have been migrated to HGPG1 GitHub during the broader Manus-era teardown. Worth a check via Vercel project list filter on the `leads.homegrownpropertygroup.com` domain attachment.
- **Where's the lead data?** PropStream-imported leads need to live somewhere — Supabase project, Manus-internal DB, or the FUB CRM proper. Should be findable by checking which Supabase project is wired to the live deployment.
- **What's the call mechanism?** Twilio Voice? Direct dialer? Power dialer integration? This matters because Twilio is fully deprecated at HGPG; if this depended on Twilio, it was already broken from the day Twilio died.
- **What happened to the leads?** Did they get tagged in FUB? Are they sitting in a separate database orphaned? Is anyone still reaching out to them ad-hoc outside the system?

## Decision pending

Three plausible paths when this gets revisited:

1. **Decommission cleanly** — domain removed, Vercel project deleted, any associated Supabase data archived or deleted. The PropStream subscription itself can be cancelled if we're not pulling new lead lists. Cleanest path if outbound calling isn't a priority.
2. **Migrate + revive** — pull code into HGPG1, update for current stack (no Twilio, current Supabase project structure, etc.), wire to LoopMessage or back to email-only, and resume outreach. Multi-day rebuild.
3. **Move leads, kill the system** — extract whatever PropStream leads are still useful into FUB with proper tags + opt-in handling, then decommission the standalone caller. Hybrid path.

## Pickup notes for next session

- This file is a placeholder, not a spec. When this project gets reactivated, the first session needs a discovery pass: find the code, find the data, find what state the leads are in, then propose a plan.
- The PropStream subscription itself may still be billing — worth checking if there's an active subscription to cancel if we're not using it
- Don NOT confuse this with `projects/fub-ai-agent.md` (the in-TM agent) or the decommissioned Manus suite at `archive/fub-agent-handoff.md`. All three are distinct. PropStream caller is its own thing.
- Compliance reminder: NCREC Rule 58A .0104 prohibits cold-calling new prospects without a license. If this gets revived, route the actual dialing through licensed agents only or restructure as a non-call outreach (text/email).
