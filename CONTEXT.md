# HGPG Context (Brain)

Last updated: 2026-05-14

## Who

**Brian McCarron** - Broker/Owner, Home Grown Property Group (HGPG), operating under Real Broker LLC. Markets: South Charlotte, Fort Mill, Indian Land, Waxhaw, plus surrounding Mecklenburg, Union, York, Lancaster counties (NC + SC). Carolinas resident 20+ years.

See `team.md` for full roster.

## Personal

- Investment property: 5022 Cressingham Dr, Indian Land, SC (STR, Airbnb listing 1461030574445690459)
- Recently helped brother-in-law evaluate a home purchase in Leola, PA
- Interests: home improvement projects (deck drainage, garage ceiling hoists)

## What's active right now

- **CMA Engine** (cma.homegrownpropertygroup.com) - active build, see `projects/cma-engine.md` (+ `projects/cma-engine-bugs-2026-05-08.md` for tracked bug pass)
- **Transaction Manager** (closings.homegrownpropertygroup.com) - lifecycle tool, see `projects/transaction-manager.md`
- **FUB AI Agent** - large active build inside the TM repo (closings.../agent surface). See `projects/fub-ai-agent.md`. Currently dev-on, `agent_enabled=false` in production until launch.
- **TC Concierge** (tc.homegrownpropertygroup.com) - intake, live with Don running real deals
- **Brain App** (brain.homegrownpropertygroup.com) - live; GitHub App auth shipped 2026-05-14, new `/api/external/commit` endpoint lets Claude push to any HGPG1 repo. See `projects/brain-app.md`.
- **Buyer Alerts** - in build, see `projects/buyer-alerts.md`
- **Buyers Guide** (buyersguide.homegrownpropertygroup.com) - React + Vite, see `projects/buyers-guide.md`
- **Sellers Guide** (sellersguide.homegrownpropertygroup.com) - Meta Pixel + CAPI live, see `projects/sellers-guide.md`
- **Charlotte New Construction** (newconstruction.homegrownpropertygroup.com) - see `projects/charlotte-new-construction.md` plus three phone-capture / speed-to-lead workstreams (`new-construction-phone-capture.md`, `new-construction-phone-capture-rate-review.md`, `new-construction-sms-speed-to-lead.md`)
- **Listing Report Portal** (reports.homegrownpropertygroup.com) - see `projects/listing-report-portal.md`. Old "blocked on GitHub auth" issue resolved by GitHub App migration on 2026-05-14; Claude can now commit to this repo directly.
- **Claude skills** - five skills shipped May 1 (objection-handler, referral-request-writer, showing-feedback-summarizer, offer-comparison-analyzer, market-update-writer). See `projects/claude-skills.md`.
- **Smaller active builds:** TM $395 fee toggle (`tm-395-fee-toggle.md`), DocuSign migration (`docusign-migration.md`), Incentives Funnel (`incentives-funnel.md`), NeverBounce email validation (`neverbounce-validation.md`), PropStream caller (`propstream-caller.md`), Signature Marketing (`signature-marketing.md`), South Charlotte Report content brand (`south-charlotte-report.md`), Team Dashboard (`team-dashboard.md`), Team Photo Sync (`team-photo-sync.md`), Team Tools (`team-tools.md`), Main Site SEO (`main-site-seo.md`), Buyers Guide Manus migration (`buyers-guide-manus-migration.md`)

## Recently completed

- **GitHub App migration (2026-05-14)** - replaced all PATs with HGPG Brain Commit GitHub App; new `/api/external/commit` endpoint live; Claude can now push to any HGPG1 repo from any device. See `SESSION-HANDOFF.md` and `projects/brain-app.md`.
- IDX Broker migration (Ylopo cancelled, Showcase IDX cancelled, FUB lead routing done)
- Main site SEO push (mobile PageSpeed 96/100)
- Buyers guide migration from Manus to React + Vite
- Sellers guide rebrand to brand colors

## Known blockers / pending

See `SESSION-HANDOFF.md` for current scratchpad.

- Sherlock 403 on Transaction Manager (likely API key scope)
- .net Google Workspace migration to .com (do not proactively remind)

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
