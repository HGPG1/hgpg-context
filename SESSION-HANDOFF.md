<!-- Last Updated: 2026-05-12 -->

# SESSION-HANDOFF

## Where we are

Two active threads in flight:

1. **Buyer Alerts Session 1 — 🟡 PARKED** on LoopMessage env var blocker (see below)
2. **FUB AI Agent — 🟢 Operational** (session 10 shipped + dashboard live, see below)

Both threads can resume independently.

---

## Thread A: Buyer Alerts S1 — 🟡 PARKED

### Status

Code shipped + merged to main on `hgpg-team-dash` (`b0ff809`). Brain doc complete at `projects/buyer-alerts.md`. DB schema applied to `wdheejgmrqzqxvgjvfee` (buyer_criteria + buyer_alerts, RLS, dedup index).

Five env vars added to Vercel project `hgpg-team-dash`:
- ANTHROPIC_API_KEY
- SUPABASE_SERVICE_ROLE_KEY
- HGPG_CORE_SUPABASE_URL
- HGPG_CORE_SUPABASE_SERVICE_ROLE_KEY
- CRON_SECRET

### Why parked

LoopMessage integration in the buyer-alerts cron route uses the **wrong env var names and wrong endpoint** vs TM's working pattern:

- buyer-alerts code expects: `LOOPMESSAGE_API_KEY` + `LOOPMESSAGE_SECRET` + `server.loopmessage.com`
- TM uses: single `LOOP_API_KEY` + `a.loopmessage.com` + raw key in Authorization header (no Bearer)

Patch already drafted (see `lib/loopBridge.ts` in TM for the canonical pattern). Replace `sendLoopMessage` function in `src/app/api/cron/buyer-alerts/route.ts` to match TM byte-for-byte.

Even after patch, the `LOOP_API_KEY` value in TM Vercel is flagged **Sensitive** and cannot be revealed to copy over. Options to resolve:

1. **Regenerate in LoopMessage dashboard** (~5 min, paste new value into BOTH TM and team-dash, redeploy both) — fastest
2. **Look in 1Password / Notes** for stored key
3. **Migrate to internal_secrets pattern** (~15 min refactor, both projects read from HGPG Core Supabase table — eliminates Sensitive-flag pain forever)

### Pick-up steps

1. Apply the LoopMessage patch (one function replacement in `src/app/api/cron/buyer-alerts/route.ts`)
2. Resolve the LOOP_API_KEY value (path 1, 2, or 3 above)
3. Add `LOOP_API_KEY` to team-dash Vercel env vars
4. Redeploy team-dash with "Use existing Build Cache" UNCHECKED
5. Smoke test:
   - Create a buyer at `team.homegrownpropertygroup.com/buyers`
   - Click "Run match now" — should return matches
   - Manual cron trigger: `curl -X GET https://team.homegrownpropertygroup.com/api/cron/buyer-alerts -H "Authorization: Bearer $CRON_SECRET"`
   - Verify iMessage arrives on Brian's phone

### Critical context

- Buyer Alerts lives INSIDE Team Dashboard repo as Tab 3 (`/buyers`), NOT a separate app
- DB tables live on `wdheejgmrqzqxvgjvfee` (Listing Reports + MLS)
- Cross-project read of `team_members.phone` on `ioypqogunwsoucgsnmla` (HGPG Core) requires service role key for that project
- LLM parser uses `claude-sonnet-4-6`, JSON-only system prompt, one retry, fallback to notes-only on second failure
- Matcher window: only `Active` listings with `modification_timestamp` within last 24 hours
- Dedup: unique index on `(criteria_id, listing_key)` — `recordMatch` returns `{created: false}` on conflict
- `property_types`, `must_have_features`, `must_avoid` are advisory only in S1 (surfaced in match_reason, not filtered) — Session 2 problem

---

## Thread B: FUB AI Agent — 🟢 Operational

Session 10 (2026-05-12) shipped + addendum. Operational dashboard live at `/agent/ops`, scoring pool backfilled +49 warm, onlyUnscored param fixed. Shared 3-tab nav strip across all agent screens (Overview/Queue/Ops) AND a full user guide section in `/dons-guide` for Don. **`agent_enabled=true`** — running for sustained operation at `daily_send_cap=10` with manual approve.

### Today's TM commits

- 956f1f9 — operational dashboard at `/agent/ops` (3 new files)
- 46d71d0 — dashboard fixes (Pool Health row cap + Activity Feed empty state)
- 7fc2fce — scoring fixes (onlyUnscored param + pagination on exclude-IDs query)
- e0ba110 — shared 3-tab nav strip via `components/AgentNav.tsx`
- (latest) — Dons guide section 16: The FUB AI Agent

### Today's brain commits

- 9350656 — session 10 entry in `projects/fub-ai-agent.md`
- 30b4893 — SESSION-HANDOFF session 10 close
- 7ae02dd — `docs/fub-agent-user-guide.md`
- edbdcfe — session 10 addendum in `projects/fub-ai-agent.md`

### Live agent state

- `agent_enabled = true` ✅
- `daily_send_cap = 10`
- `auto_below_threshold = false` (manual approve only)
- Thresholds: hot=40, warm=30, llm_confidence_floor=0.4
- Pool: 5,373 eligible / 0 hot / 52 warm / 981 cold / 4,340 unscored

### Where the agent lives in the UI

- `/agent` — Overview (score distribution + top 50 leads)
- `/agent/queue` — Approval queue
- `/agent/ops` — Live monitoring dashboard (7 panels, 30s auto-refresh)

All three share `components/AgentNav.tsx` tab strip. Dashboard also linked from Dons Guide §16 at `/dons-guide#fub-agent`.

### Session 11 priorities for the agent

1. **Watch the agent live for a few days.** Real Brian + Don usage data. Approve rate? Reject reasons? Voice complaints?
2. **Optional:** continue scoring backfill with `onlyUnscored=true` (~$10-15, ~100 expected warm conversions)
3. **Decide on `auto_below_threshold` flip** once you trust the queue UI workflow
4. **Hot tier templates** if any leads start hitting score >=40

### Critical context (FUB Agent)

- `agent_enabled` blocks BOTH cron AND manual approve via `outboundGate`. Flip via Supabase MCP or the queue UI toggle.
- FUB Automation 2.0 delivers email as plaintext. NEVER use `<br>` in templates — real newlines only.
- FUB UI Merge Fields dropdown produces `%snake_case%` tokens. Use the dropdown, don't type.
- Test rig: FUB person 27764 (Brian in Sphere, pond 9 = excluded from agent).
- gerardmarmo@yahoo.com (FUB 23552): Brian in active manual conversation. Reject any agent drafts.
- For pool backfills: `POST /api/agent/score` body `{"limit": N, "onlyUnscored": true}` to score N leads that have NEVER been scored.
- **Supabase JS has a 1000-row default cap on unbounded selects.** Always paginate or use count head:true.
- **PostgREST embed syntax silently returns null without a FK.** When embedding a related table, verify the FK exists or fetch + merge in JS.

### Two user guides

- **Dons guide** at `/dons-guide#fub-agent` — operational, warm voice, for daily TC use
- **Developer reference** at `docs/fub-agent-user-guide.md` in brain repo — terser, technical

### FUB Agent backlog (multi-session, priority order)

- Hot tier templates (score >= 40)
- iMessage seller variants
- Cooldown re-touch templates (second-attempt copy)
- More buyer/seller template variants (only viewed_listing_recent has variants)
- Backfill remaining 3,340 unscored leads with `onlyUnscored=true`
- FUB UI custom fields 157-162 visibility to admins-only
- Normalize hideIfEmpty across fields 157 vs 160/161/162 (cosmetic)
- Stale tag cleanup on person 27764
- Drop body column after deprecation window
- v2: brokerage oversight expansion (Ashley/Brenda/Taylor + Don unified queue)
- v2: tier-based behavior, smart channel pick, production inbound classifier

---

## Locked queue across all threads

1. ⏳ **Buyer Alerts S1 unblock** (LoopMessage env + smoke test) — small, but blocking
2. 🎯 **Buyer Alerts Session 2** (capture UX polish, agent management, alert history, manual test button)
3. 🎯 FUB Agent monitoring + tuning (passive, ongoing)
4. 🎯 Team listing photo sync (~30 min, on-demand, see `projects/team-photo-sync.md`)
5. 🔍 Off-market / Expireds Finder (~5-6 hr, replaces external scraper)

### Smaller open items

- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media full sync (parked)
- Sellers guide Brian-only cleanup: ScoreCompleted Custom Conversion in Events Manager, delete QA test leads from FUB, remove "Phase 1 ads test markers"

### Parked

- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — Team Dashboard covers it)
- Meta Lead Ads → FUB native integration (only matters for Instant Form ads)
