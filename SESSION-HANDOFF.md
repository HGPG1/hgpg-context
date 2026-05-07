<!-- Last Updated: 2026-05-07 -->

# Session Handoff

## Pickup: FUB AI Agent session 5

Pick up FUB AI Agent at session 5. Read `projects/fub-ai-agent.md` for full session 4 detail before changing anything.

### Session 5 priorities (in order)

1. **Confirm Brian's FUB person ID.** Hit `GET /v1/people?email=brian@homegrownpropertygroup.com` (or whatever address Brian uses inside FUB). Capture the `id`. Needed for the smoke test in step 3.
2. **Wire FUB Automations 2.0 triggers** for the two ready tags `agent_draft_email_ready` and `agent_draft_imessage_ready`. Each automation should:
   - Trigger on the tag being added to a person
   - Read the channel-specific custom field (`customAgentDraftEmailBody` + `customAgentDraftEmailSubject` for email; `customAgentDraftIMessageBody` for iMessage)
   - Send via FUB-native email or via Sendblue for iMessage
   - Remove the trigger tag at the end so the same tag can fire again later
3. **Brian-as-client smoke test.** With Brian's own person ID seeded into `fub_agent_leads` as a test lead and a known draft staged in `pending_review`:
   - Flip `agent_enabled` to true via the queue header
   - Approve the draft from the UI; confirm `pushDraftToFUB` succeeds, custom field appears on Brian's FUB person, tag fires, automation runs, real email/iMessage lands
   - Flip `agent_enabled` back to false immediately
4. **Daily-cap flush cron at midnight ET.** Approved drafts with null `fub_pushed_at` should be re-pushed each midnight ET so a same-day cap does not strand them forever. Wire as a Vercel cron pointing at a new route, gated by `CRON_SECRET` like the other agent routes.
5. **Wire inbound classifier to optout + cooldown tables.** Currently the inbound stub at `app/api/agent/inbound/route.ts` only logs the classification. Session 5 should branch:
   - `opt_out` → insert into `fub_agent_lead_optouts`
   - `hostile` → 60-day cooldown
   - `not_now` → 30-day cooldown
   - `interested` / `irrelevant` → log only
6. **Set admins-only visibility** on the three new FUB custom fields (157, 158, 159) via the FUB UI. The customFields API does not expose visibility.

### Current system state (as of 2026-05-07 EOD session 4)

- 159 confidently classified leads
- `agent_enabled` = **false** (master kill switch, cron is no-op)
- `daily_send_cap` = 10
- 3 FUB custom fields configured (`customAgentDraftEmailBody`, `customAgentDraftEmailSubject`, `customAgentDraftIMessageBody`) — IDs 157, 158, 159
- 2 FUB tags scheduled to appear on first apply (`agent_draft_email_ready`, `agent_draft_imessage_ready`)
- 0 outbound messages have ever been sent
- Cooldown table `fub_agent_lead_cooldowns` live, currently empty
- 5-class reject taxonomy live in queue UI
- Queue header (kill switch + cap editor) live at `/agent/queue`

### Where things live

- TM repo: `HGPG1/hgpg-transaction-manager` on `main`
- Latest commit: `269c867` — session 4 ship
- Supabase: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel: TM project (`prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`)
- Brain: this repo, `projects/fub-ai-agent.md` is the canonical multi-session log

---

## Previous session: 2026-05-07 — Sellers Guide Meta Pixel + CAPI fully verified 🟢

(Full notes below preserved for context — not actionable.)

### What shipped
- **Meta Pixel + CAPI on sellers guide is production-ready and verified end-to-end**
- Path A (organic) QA: PASSED — PageView, AssessmentStarted, Lead, ScoreCompleted all fire browser + server with matching `event_id` and Meta-side dedup confirmed
- Path B (Meta bypass) QA: PASSED — phone required, 6-digit verify skipped, FUB lead lands with `meta-bypass` tag and ALL 7 UTM/click custom fields populated
- `META_TEST_EVENT_CODE` env var removed from Vercel + redeployed — production traffic now flows to real Events Manager dashboards (not Test Events tab)
- Production deploy on commit `8ea82cc`

### Bug found + fixed mid-session
- `api/fub-lead.js` was sending FUB custom field **labels** (e.g., `"UTM Source"`) as object keys instead of FUB API **names** (e.g., `customUTMSource`). FUB silently drops unknown keys, so all 7 fields had been failing silently the whole time.
- Confirmed correct API names via `GET /v1/customFields`:
  - `customUTMSource`, `customUTMMedium`, `customUTMCampaign`, `customUTMContent`, `customUTMTerm`, `customFacebookClickID`, `customGoogleClickID`
- Patched in commit `8ea82cc` on `main`. Also added `X-System` / `X-System-Key` headers and a `DEBUG_FUB=1` env flag for surfacing FUB error bodies in API responses when needed.

### Carryover for the sellers-guide meta work (still open)

**Create `ScoreCompleted` Custom Conversion in Events Manager** (~5 min)
- Direct URL: `https://business.facebook.com/events_manager2/list/dataset/861295553661596` → Custom Conversions → Create
- Settings:
  - Name: `Sellers Guide - Score Completed`
  - Description: `User finished home selling score assessment`
  - Data source: HGPG — Sellers Guide
  - Action Source: Website
  - **Event: `ScoreCompleted`** (NOT Lead — Lead would double-count form submissions)
  - Rules: URL contains `home-selling-score`
- Once created, takes 24-48 hours of data to warm up before usable as an ad optimization goal.

**Cleanup tasks (low priority)**
- Delete QA test leads from FUB:
  - `qa-may7-fields-test@hgpg-test.com`
- Remove "Phase 1 ads test markers" in code

### Next big initiative: Meta Pixel + CAPI rollout to remaining sites

Bring the proven sellers-guide pattern to:
- **Transaction Manager** (closings.homegrownpropertygroup.com)
- **Marketing analyzer** (which site/repo — confirm next session)
- **Signature** (signature.homegrownpropertygroup.com)

Playbook: `META-PIXEL-CAPI-PLAYBOOK.md` in sellers guide repo root. Estimated ~30 min/site.
