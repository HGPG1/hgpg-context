<!-- Last Updated: 2026-05-12 -->

# SESSION-HANDOFF

## Where we are

**Session 10 (2026-05-12) shipped + addendum.** Operational dashboard live at /agent/ops, scoring pool backfilled +49 warm, onlyUnscored param fixed so future backfills are clean. Plus post-close: shared 3-tab nav strip across all agent screens (Overview/Queue/Ops) AND a full user guide section in /dons-guide for Don. **agent_enabled=true** - the FUB AI Agent is running for sustained operation at daily_send_cap=10 with manual approve.

Status: 🟢 Operational + monitored + documented. Live for week 1 ramp.

## Todays commits (TM)

- 956f1f9 - operational dashboard at /agent/ops (3 new files: /agent/ops/page.tsx, /agent/ops/DashboardClient.tsx, /api/agent/dashboard/route.ts)
- 46d71d0 - dashboard fixes: Pool Health row cap + Activity Feed empty state
- 7fc2fce - scoring fixes: onlyUnscored param + pagination on exclude-IDs query
- e0ba110 - shared 3-tab nav strip across all agent screens (Overview/Queue/Ops via components/AgentNav.tsx)
- (latest) - dons guide section 16: The FUB AI Agent (~96 lines added to app/dons-guide/content.ts, also TOC entry + cheat sheet row + cron schedule update + renumbered feedback to §17)

## Todays brain commits

- 9350656 - session 10 entry in projects/fub-ai-agent.md
- 30b4893 - SESSION-HANDOFF session 10 close
- 7ae02dd - docs/fub-agent-user-guide.md (standalone markdown reference)
- edbdcfe - session 10 addendum in projects/fub-ai-agent.md (nav + guide section)
- (this commit) - SESSION-HANDOFF session 10 addendum

## TM repo state

- Branch: main
- HEAD: latest (after dons-guide commit)
- Deploy: closings.homegrownpropertygroup.com READY
- Working tree clean

## Live agent state

- agent_enabled = true ✅
- daily_send_cap = 10
- auto_below_threshold = false (manual approve only)
- hot_threshold = 40, warm_threshold = 30, llm_confidence_floor = 0.4
- Pool: 5,373 eligible / 0 hot / 52 warm / 981 cold / 4,340 unscored

## Where the agent lives in the UI

- /agent — Overview (read-only v1 dashboard, score distribution + top 50 leads)
- /agent/queue — Approval queue
- /agent/ops — Live monitoring dashboard (7 panels, 30s auto-refresh)

All three pages now share a tab strip at the top with active highlighting via `components/AgentNav.tsx`. The dashboard is also linked from Dons Guide §16 at /dons-guide#fub-agent.

## Pick up here (session 11 priorities)

1. **Watch the agent live for a few days.** Real Brian + Don usage data. What is the approve rate? Reject reasons? Voice complaints?
2. **Optional: continue scoring backfill** with onlyUnscored=true to drain the remaining 3,340 unscored leads (~$10-15, ~100 expected warm conversions)
3. **Decide on auto_below_threshold flip** once you trust the queue UI workflow
4. **Hot tier templates** if any leads start hitting score >=40

## Critical context (carry forward)

- agent_enabled blocks BOTH cron AND manual approve via outboundGate. Flip via Supabase MCP or the queue UI toggle.
- FUB Automation 2.0 delivers email as plaintext. NEVER use <br> in templates - real newlines only.
- FUB UI Merge Fields dropdown produces %snake_case% tokens. Use the dropdown, dont type.
- Test rig: FUB person 27764 (Brian in Sphere, pond 9 = excluded from agent).
- gerardmarmo@yahoo.com (FUB 23552): Brian in active manual conversation. Reject any agent drafts.
- Template paragraph sweep query lives in projects/fub-ai-agent.md "Session 8" entry.
- Variant naming convention: v1.warm.scenario_name.channel (base) + .v2/.v3/... for variants.
- **For pool backfills:** use POST /api/agent/score with body {"limit": N, "onlyUnscored": true} to score N leads that have NEVER been scored. Dont use the default backfill - it re-scores leads we already have.
- **Supabase JS has a 1000-row default cap on unbounded selects.** Always paginate or use count head:true.
- **PostgREST embed syntax silently returns null without a FK.** When embedding a related table, verify the FK exists or fetch + merge in JS.

## Two user guides

- **Dons guide** at /dons-guide#fub-agent — operational, warm voice, for daily TC use
- **Developer reference** at docs/fub-agent-user-guide.md in brain repo — terser, technical

## Backlog (multi-session, in priority order)

- Hot tier templates (score >= 40)
- iMessage seller variants
- Cooldown re-touch templates (second-attempt copy)
- More buyer/seller template variants (only viewed_listing_recent has variants)
- Backfill remaining 3,340 unscored leads with onlyUnscored=true
- FUB UI custom fields 157-162 visibility to admins-only
- Normalize hideIfEmpty across fields 157 vs 160/161/162 (cosmetic)
- Stale tag cleanup on person 27764
- Drop body column after deprecation window
- v2: brokerage oversight expansion (Ashley/Brenda/Taylor + Don unified queue)
- v2: tier-based behavior, smart channel pick, production inbound classifier
