<!-- Last Updated: 2026-05-12 -->

# SESSION-HANDOFF

## Where we are

**Session 10 (2026-05-12) shipped.** Operational dashboard live at /agent/ops, scoring pool backfilled +49 warm, onlyUnscored param fixed so future backfills are clean. **agent_enabled=true** - the FUB AI Agent is now running for sustained operation at daily_send_cap=10 with manual approve.

Status: 🟢 Operational + monitored. Live for week 1 ramp.

## Today's commits (TM)

- 956f1f9 - operational dashboard at /agent/ops (3 new files: /agent/ops/page.tsx, /agent/ops/DashboardClient.tsx, /api/agent/dashboard/route.ts)
- 46d71d0 - dashboard fixes: Pool Health row cap (Supabase JS 1000-row default) + Activity Feed empty state (missing FK on embed)
- 7fc2fce - scoring fixes: onlyUnscored param + pagination on exclude-IDs query

## Today's brain commits

- 9350656 - session 10 entry in projects/fub-ai-agent.md
- (this commit) - session 10 SESSION-HANDOFF update

## TM repo state

- Branch: main
- HEAD: 7fc2fce
- Deploy: closings.homegrownpropertygroup.com READY
- Working tree clean

## Live agent state

- agent_enabled = true ✅
- daily_send_cap = 10
- auto_below_threshold = false (manual approve only)
- hot_threshold = 40, warm_threshold = 30, llm_confidence_floor = 0.4
- Pool: 5,363 eligible / 0 hot / 52 warm / 971 cold / 4,340 unscored

## Open queue (~14 drafts in flight at last check)

Manual approve required on every draft.

Buyer queue: Anthony (clean), Sam (clean), Chris Smith (clean), plus ~3 v3 drafts (Gerard - reject, Seanna, others)
Seller queue: John, Jay, Tamela (post-template-fix clean), Rachel, Jason, Leigh, Tracy (newly classified seller)
Plus Jesse Hernandez dupe to reject (id 23)

## Pick up here (session 11 priorities)

1. **Watch the agent live for a few days.** Real Brian + Don usage data. What's the approve rate? Reject reasons? Voice complaints?
2. **Optional: continue scoring backfill** with `onlyUnscored=true` to drain the remaining 3,340 unscored leads (~$10-15, ~100 expected warm conversions)
3. **Decide on auto_below_threshold flip** once you trust the queue UI workflow
4. **Hot tier templates** if any leads start hitting score ≥40

## Critical context (carry forward)

- agent_enabled blocks BOTH cron AND manual approve via outboundGate. Flip via Supabase MCP or the queue UI toggle.
- FUB Automation 2.0 delivers email as plaintext. NEVER use `<br>` in templates - real newlines only.
- FUB UI Merge Fields dropdown produces `%snake_case%` tokens. Use the dropdown, don't type.
- Test rig: FUB person 27764 (Brian in Sphere, pond 9 = excluded from agent).
- gerardmarmo@yahoo.com (FUB 23552): Brian in active manual conversation. Reject any agent drafts.
- Template paragraph sweep query lives in projects/fub-ai-agent.md "Session 8" entry.
- Variant naming convention: `v1.warm.scenario_name.channel` (base) + `.v2`/`.v3`/... for variants.
- **For pool backfills:** use `POST /api/agent/score {"limit": N, "onlyUnscored": true}` to score N leads that have NEVER been scored. Don't use the default backfill - it re-scores leads we already have.
- **Supabase JS has a 1000-row default cap on unbounded selects.** Always paginate or use count head:true.
- **PostgREST embed syntax silently returns null without a FK.** When embedding a related table, verify the FK exists or fetch + merge in JS.

## Two dashboards

- **/agent** - read-only score distribution histogram + top-50 leads table. The lead inspector.
- **/agent/ops** - 7-panel operational dashboard, 30s auto-refresh. The daily-glance live view.
- Queue lives at **/agent/queue**

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
