<!-- Last Updated: 2026-05-07 -->

# Session Handoff

## Pickup: FUB AI Agent session 6

Pick up FUB AI Agent at session 6. Smoke test should be complete by now — confirm with Brian. Read `projects/fub-ai-agent.md` for the session 5 detail before changing anything.

### Session 6 priorities (in order)

1. **Review smoke-test outcome with Brian.**
   - Did the manual SQL flow at `scripts/session-5-smoke-test.sql` complete cleanly?
   - Did Brian receive the iMessage on +1 803 902 3700? Did email follow?
   - Are FUB Automations 2.0 actually live for both ready tags (Viktor)?
   - Pull `fub_agent_log` rows tagged `fub_push_success` / `fub_push_failure` from the smoke test window — anything weird?
   - **If anything is off, halt and triage; do not proceed to step 2.**
2. **First real outbound batch (only if smoke test was clean).**
   - Plan a small (e.g. 3-5 lead) initial run under the daily cap of 10
   - Pick the leads from the queue UI by hand; do not auto-batch
   - Brian flips `agent_enabled` true, approves them one by one, watches the FUB-side delivery + responses
   - Observe for 24-48h; revisit cap, cooldowns, classifier behavior
3. **"Interested" inbound surfacing in queue UI.**
   - Right now `inbound_interested` only logs. Build a small section in `/agent/queue` listing recent interested replies (joined to lead) so Brian can act on them.
   - Schema may need a small denormalized view or a new `fub_agent_inbound_replies` table; investigate before changing.
4. **Week 1 metrics dashboard.**
   - Pushes/day vs cap
   - Opt-out rate (rows in `fub_agent_lead_optouts` over messages pushed)
   - Inbound classification breakdown
   - Active cooldowns by reason
5. **Autonomous-mode evaluation criteria.**
   - What would have to be true (numerically + behaviorally) to flip `auto_below_threshold = true`?
   - Document on `projects/fub-ai-agent.md` so we don't move that gate by feel

### Current system state (as of 2026-05-07 EOD session 5)

- `agent_enabled` = **false** (master kill switch, all crons no-op)
- `daily_send_cap` = 10
- `fub_agent_lead_cooldowns_reason_check` enum now includes `hostile_reply` and `not_now_reply` (added in session 5)
- `fub_agent_lead_optouts` empty
- `fub_agent_lead_cooldowns` empty
- `fub_agent_message_drafts` 0 with `fub_pushed_at` non-null
- 0 outbound messages have ever been sent
- Cron `/api/cron/agent-daily-flush` registered in vercel.json on `1 5 * * *` UTC. No-op while `agent_enabled = false`.
- Smoke test SQL artifact lives at `hgpg-transaction-manager/scripts/session-5-smoke-test.sql`
- Brian's FUB person id = **27764**, currently excluded (pond:brian_excluded_contacts). Smoke test temporarily removes exclusion + restores in cleanup.
- 3 FUB custom fields configured (`customAgentDraftEmailBody`, `customAgentDraftEmailSubject`, `customAgentDraftIMessageBody`)
- 2 FUB tags scheduled to appear on first apply (`agent_draft_email_ready`, `agent_draft_imessage_ready`)
- FUB Automations 2.0: built? coordinate with Viktor before assuming live
- FUB UI visibility on custom fields: still pending the FUB UI step (carryover from session 4)

### Where things live

- TM repo: `HGPG1/hgpg-transaction-manager` on `main`
- Latest commit: `eb9089d` — session 5 ship
- Supabase: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel: TM project (`prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`)
- Brain: this repo, `projects/fub-ai-agent.md` is the canonical multi-session log

### Blockers / dependencies

- **FUB Automations 2.0** — Viktor's lane. Without them, custom-field writes + tag applies happen but no actual outbound message lands. Smoke test cannot complete without these.
- **`FUB_AGENT_INBOUND_SECRET`** must be set in Vercel before the inbound classifier can be invoked end-to-end. Until set, `/api/agent/inbound` returns 503.

---

## Previous session: 2026-05-07 — Sellers Guide Meta Pixel + CAPI fully verified 🟢

(Full notes below preserved for context — not actionable.)

### What shipped
- **Meta Pixel + CAPI on sellers guide is production-ready and verified end-to-end**
- Path A (organic) QA: PASSED — PageView, AssessmentStarted, Lead, ScoreCompleted all fire browser + server with matching `event_id` and Meta-side dedup confirmed
- Path B (Meta bypass) QA: PASSED — phone required, 6-digit verify skipped, FUB lead lands with `meta-bypass` tag and ALL 7 UTM/click custom fields populated
- `META_TEST_EVENT_CODE` env var removed from Vercel + redeployed — production traffic now flows to real Events Manager dashboards (not Test Events tab)
- Production deploy on commit `8ea82cc`

### Carryover for the sellers-guide meta work (still open)

**Create `ScoreCompleted` Custom Conversion in Events Manager** (~5 min)
- Direct URL: `https://business.facebook.com/events_manager2/list/dataset/861295553661596` → Custom Conversions → Create
- Settings:
  - Name: `Sellers Guide - Score Completed`
  - Event: `ScoreCompleted` (NOT Lead — Lead would double-count form submissions)
  - Rules: URL contains `home-selling-score`
- Once created, takes 24-48 hours of data to warm up before usable as an ad optimization goal.

**Cleanup tasks (low priority)**
- Delete QA test leads from FUB: `qa-may7-fields-test@hgpg-test.com`
- Remove "Phase 1 ads test markers" in code

### Next big initiative: Meta Pixel + CAPI rollout to remaining sites

Bring the proven sellers-guide pattern to:
- **Transaction Manager** (closings.homegrownpropertygroup.com)
- **Marketing analyzer** (which site/repo — confirm next session)
- **Signature** (signature.homegrownpropertygroup.com)

Playbook: `META-PIXEL-CAPI-PLAYBOOK.md` in sellers guide repo root. Estimated ~30 min/site.
