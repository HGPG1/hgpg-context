## Last session: 2026-05-20 (PM) — IDXRE-B2 activation, smart lists shipped, canary + bulk-apply delayed to Thursday 9am 🟢

### Context
Phase 2 (B2) of IDXRE-2026-04 re-engagement campaign moved from planning into live send. Picked up where IDXRE-2026-04 was parked after Phase 1 contamination audit: KTS-pause confirmed at zero, automations 336 (Sellers) + 337 (Buyers) verified built by Viktor with templates 1166/1167 wired, but everything still INACTIVE. This session got it live.

### What shipped

**Smart lists built and verified clean:**
- **List 81 — IDXRE-B2-Sellers-Target** (894 leads after v2 exclusions). Pond 16 + stage Expired/Withdrawn + 12 exclusion tags.
- **List 82 — IDXRE-B2-Buyers-Target** (364 leads after v2 exclusions). Pond 16 + stage IS NOT Expired/Withdrawn + same 12 exclusion tags.
- v1 → v2 audit caught contamination: Barry Falls DNC slipped through, 3 DO_NOT_CALL leads in buyer list, ~12 parked Top25 tier leads. v2 added: DO_NOT_CALL, Do Not Contact, Top25-DNCFlagged, bare Unsubscribed, Top25-Hot/AIFlagged/Engaged. Per Brian's call, AI tags (AI_ENGAGED, AI_VOICE_NEEDS_FOLLOW_UP, AI_NEEDS_FOLLOW_UP) intentionally NOT excluded.
- PDF parameters doc shipped: `/mnt/user-data/outputs/IDXRE-B2_SmartList_Parameters_v2.pdf`

**Automations 336/337 flipped ACTIVE in FUB UI by Brian.** Architecture confirmed (Viktor's design): manual trigger by intent, NOT auto-enrollment from smart list. Single-step cadence: Send Email → Add Tags (`IDXRE-2026-05-19-B2`, `IDXRE-B2-Sent`) → done. `IDXRE-B2-Sent` serves as self-suppression marker so leads can't be double-fired.

**Canary roster sent (15 leads):**
- 10 manual one-at-a-time enrollments via FUB UI (5 sellers + 5 buyers): Norma Mack, Brian Barnhardt, Zo Pui, Daniel Stevens, Georgekutty Scaria; Molly Waldron, Michelle White, Javier Bucio, Jennifer Hammond, Lindsey Christian. Jeffrey Hall (30972) pulled from roster — turned out he's a realtor who'd found Brian on Realtor.com, trashed. Hammond (HomeLight source) swapped in.
- 5 additional via Mass Action bulk-test in FUB UI. Brian verified emails fired.
- All 15 sends confirmed (replied tag landed, activity feed shows Email sent event).

**Bulk apply queued, delayed to Thursday 2026-05-21 9:00 ET** for the remaining 1,238 leads (884 sellers + 354 buyers minus the 15 canary). Step 1 of automation delayed to morning send window for higher inbox engagement.

### Brain writes this session
- `SESSION-HANDOFF.md` — this entry
- (No project file updates this session — IDXRE-B2 progress lives here until campaign closes out)

### Lessons / patterns

- **Mass Action in FUB UI DOES fire automations.** This contradicts the previously documented gotcha in `projects/transaction-manager.md` / project instructions: *"Mass Actions do NOT fire automations - re-running automations requires re-firing trigger or cloning with new conditions."* Brian empirically tested with 5 leads — emails fired, tags landed. Memory needs updating. Possibilities: behavior changed since gotcha was logged, gotcha was specific to a different automation type (Action Plans not Automations 2.0), or gotcha was wrong. Empirical ground truth wins.
- **`addPersonToAutomation` MCP returns 403.** Same gating as `listAutomations` / `getAutomation` — endpoint requires registered developer X-System credentials. Cannot be used from chat sessions. For high-volume enrollment going forward, build Python script in `~/Documents/hgpg-transaction-manager/fub_cleanup/` with X-System + X-System-Key headers (pattern from existing `fub_bulk_tag.py`, `fub_legacy_pause.py`).
- **`listSmartLists` MCP returns metadata only**, no array body. Broken wrapper. Use `listPeople` with `smartListId=N` parameter instead — works.
- **Templates created via API default to `createdById=12` (Owner Account).** Templates 1166 and 1167 were both created by Viktor via API and inherited owner=12 instead of Viktor's userId. Non-blocking, but worth fixing the pattern next time a template gets created via API. The `isShared: false` API default means other FUB users can't see the template until shared manually in UI — verified both 1166/1167 are now `isShared: true`.
- **GitHub `raw.githubusercontent.com` caches aggressively** (5-15 min stale). Confirmed today: raw URL returned 2026-05-06 cached version while GitHub API (api.github.com/.../contents/...) returned current state with sha. For brain reads in time-sensitive sessions, use GitHub API blob endpoint or the brain-app `/api/external/read` endpoint, never raw URL.

### Open / parked from this session
- **Memory update needed:** Mass Action DOES fire Automations 2.0. Overrules previously documented gotcha. Brian acknowledged in-session.
- **Template ownership cleanup (templates 1166, 1167):** createdById=12 should be Viktor or Brian's userId. Non-blocking, cosmetic.
- **Python `fub_enroll_canary.py` script:** Not built yet — the 1,238 bulk apply now goes through FUB UI Mass Action since that path works. Script still worth building for future campaigns where automation needs X-System creds path (e.g. if Mass Action stops firing automations, or for scheduled cron-style enrollment).
- **SG-2026 Sellers Guide automations:** Partial state confirmed. 8 templates built (IDs 1158-1165), only 3 wired to automations (1162, 1164, 1165 each have 1 attached). 5 templates have zero automations. Test stub template 1157 `TEST-DELETE-SG2026-temp` needs deletion. Slack to Viktor sent in-session, awaiting his full API verification (he offered).
- **Viktor follow-up received late-session (post-bulk-apply queue):**
  - Meta Ads Variant E + Variant D NC ad set: live, 3 placements uploaded. Brian had pre-approved before launch.
  - FUB Automations 336/337: built INACTIVE (already known, Brian flipped them ON in-session).
  - SG-2026 Sellers Guide automations: status unclear from Viktor's side, he offered a full API check (acknowledged in-session, partial check already done).
  - FUB Variant D Automation (NC Incentives): built 2026-05-11, triggers on `utm_content = variant_d_localpride`, matches A/B/C structure.

### Pickup for next session

- **Thursday 2026-05-21 ~10am ET first check:** Open IDXRE-B2-Sent tag in FUB, count should hit ~1,253 leads. Watch Inbox for replies, watch for bounce wave.
- **Watch the 5 special-case canary leads in particular:**
  - Brenda Le (26291) — AI_ENGAGED tag, confirms B2 plays nice with AI-touched leads
  - Deusilene Moreira (26299) — Taylor List + Amanda Morgan tags, confirms Taylor-assigned leads can receive Brian-signed email without confusion
  - Zo Pui (30960) — FSBO/Seller variant stage
  - Georgekutty Scaria (30745) — Canceled variant stage
  - Jennifer Hammond (27837) — HomeLight source, only HomeLight in canary
- **SG-2026 follow-up with Viktor:** Confirm 5 unwired templates get automations attached, or delete if not needed. Test stub 1157 deletion.
- **Update `projects/transaction-manager.md` or wherever the Mass Action gotcha lives** to reflect Brian's empirical override.
- **Build `fub_enroll_canary.py`** if future campaigns need higher-control enrollment than Mass Action allows.

### Numbers state at session end
- Pond 16 active for B2: 1,258 leads in smart lists (894 sellers + 364 buyers)
- Already sent today: 15 leads (10 manual canary + 5 Mass Action test)
- Queued for Thursday 9am ET: 1,243 leads (884 sellers - 5 already test-sent + 354 buyers - depends on which 5 were tested)
- Both automations live and confirmed firing
- `IDXRE-B2-Sent` tag count baseline = 15 (will validate Thursday morning)

---
