<!-- Last Updated: 2026-05-12 -->

# FUB AI Agent

- **Status:** 🟢 Operational + monitored. Session 10 (2026-05-12) shipped operational dashboard at /agent/ops + scoring pool backfill (1,000 leads, +49 warm) + onlyUnscored param for clean future backfills. agent_enabled=true, ready for week 1 ramp at daily_send_cap=10 with manual approve.

## Session 10 — 2026-05-12 (operational dashboard + scoring backfill + onlyUnscored fix)

### What shipped

**Operational dashboard at /agent/ops (commit 956f1f9 + 46d71d0).** New screen for daily monitoring. Sits alongside the existing v1 read-only dashboard at /agent (score distribution + top-50 leads) - two screens, two purposes. The queue UI already had a `← Back to dashboard` link pointing to /agent, so the existing dashboard remains the front page and /agent/ops is the deeper operational view.

Seven panels:
1. **Pool health** - eligible_total / hot / warm / cold / unscored counts with draftable subtotal + scored % progress
2. **Daily cap (ET)** - pushed_today / cap with progress bar, red when at cap
3. **Volume (last 7 days)** - generated/approved/sent/rejected/blocked + delta vs prior 7 days
4. **Queue depth** - pending count by template scenario
5. **Template performance (last 30 days)** - total/approved/rejected/blocked/pending + approve % per scenario
6. **Top reject reasons (last 30 days)** - grouped, with empty state
7. **Recent activity feed** - last 20 events with timestamp + event_type + lead name + summarized event_data

Single endpoint `GET /api/agent/dashboard` returns all panel data in one shot. Client component auto-refreshes every 30 seconds, only when tab is visible (visibilitychange listener pauses interval when document.hidden). Visual idiom matches queue UI exactly: bg-[#F0F0F0] page, bg-white rounded-lg border-[#D1D9DF] cards, text-[#2A384C] navy, text-[#5a6b7d] secondary.

**Two post-deploy bug fixes in patch commit 46d71d0:**
- Pool Health was capped at 1,000 leads because Supabase JS defaults to a 1000-row limit on unbounded selects. Dashboard showed 1,000 eligible / 771 unscored when the real numbers are 5,363 eligible / ~3,340 unscored. Fix: use `count: exact head: true` for the eligible total, paginate the actual ID pulls in pages of 1000 with a 20-page defensive cap.
- Activity feed always showed empty because the PostgREST embed syntax `fub_agent_leads(name)` silently returned null - no FK exists from `fub_agent_log` to `fub_agent_leads`. Fix: drop the embed, collect unique fub_person_ids from the 20 activity rows, fetch lead names in one separate query, build a Map for the merge.

**Scoring pool backfill via canary approach.** Ran 5 sequential calls of limit=200, total 1,000 leads scored. Results:
- Batch 1: 19 warm of 200 (9.5%) - freshest by last_activity
- Batches 2-5: 4-9 warm each, averaging 3.8% - diminishing returns as we move into stale leads
- Total: 49 warm added to pool from 1,000 scored. ~$5 in Haiku tokens at observed cost rate
- Pool growth: 22 → 52 warm (estimated ~25 warm pre-backfill, hit 49 fresh + losses from cooldowns etc)
- 0 errors across 1,000 calls

**onlyUnscored param + pagination fix (commit 7fc2fce).** The 1,000-lead backfill surfaced that ~855 of 1,000 scoring calls re-scored leads we already had instead of growing the pool. Two bugs combined:
- Bug 1: scoreEligibleLeads order-by `last_activity` prioritizes leads with the most recent FUB activity, which heavily overlaps with leads we have already scored. Default 24h rescore window meant we burned tokens on stale-by-time but fresh-by-data leads.
- Bug 2: the recently-scored IDs query had Supabase JSs default 1000-row cap. With ~1,170 leads scored in our DB, the exclusion set was incomplete - some leads escaped the not-in filter and got re-scored anyway.

Fix: new `onlyUnscored: boolean` param on scoreEligibleLeads + the route. When true, the exclude set is ALL ever-scored leads (not just recent), so backfills pull exclusively unscored leads. Both code paths now paginate the exclude-IDs query in pages of 1000 with a 50-page defensive cap. Daily cron untouched - still uses default recent-only behavior. Verified with canary `{"limit":10,"onlyUnscored":true}` - all 10 unscored leads pulled, skipped_already_scored=2819 (full exclude set used vs ~800 before).

**Agent flipped agent_enabled=true at session close** for week 1 ramp. daily_send_cap=10 with manual approve.

### Learnings captured in session 10

1. **Supabase JS has a default 1000-row cap on unbounded selects.** Bit us twice in one session: Pool Health (returned 1000 of 5363 eligible) and scoreEligibleLeads recently-scored query (incomplete exclude set). Pattern fix: any query that pulls "all rows matching a filter" needs to either use `count: exact head: true` (if you only need the count) or paginate via `range(from, from + PAGE - 1)` in a loop. Never rely on unbounded selects.
2. **PostgREST embed syntax silently returns null without a FK.** `select('field, related_table(field)')` returns null for the embed when no foreign key relationship exists in Postgres, regardless of whether the data is queryable. Caught by the empty activity feed. Lesson: if you're not sure an FK exists, fetch the related rows in a separate query and merge in JS rather than relying on the embed.
3. **Order-by last_activity for backfills prioritizes leads we already have.** The scoring engine's default `orderBy=last_activity` is correct for the daily refresh cron (re-score the freshest leads first), but wrong for growing the pool (those leads are already scored). For backfills, onlyUnscored=true is the right tool; consider also adding orderBy=random for future backfills to spread across the population.
4. **Tail-end pool conversion rate is much lower than headline.** Freshest 200 leads converted at 9.5% warm. Trailing 800 averaged 3.8%. Final 3,340 unscored leads will likely convert at 2-4% as we move into staler territory. For pool-growth ROI math, assume 3% not 10%.
5. **Two-screen dashboard pattern works.** Original /agent (read-only inspector) + new /agent/ops (operational live view) split the use cases cleanly. Don and Brian glance at /agent/ops during the workday. /agent is the lead-inspector for deep-dives. No need to merge them.

### Followup observations (not blocking)

- The "missing FK" pattern likely affects other places in the codebase that embed `fub_agent_leads(*)` from `fub_agent_log` or similar. Worth a future sweep, but not urgent.
- The `recentlyScoredIds` -> `excludeIds` rename in scoreEligibleLeads loses some semantic meaning. If we later need to distinguish "recently rescored" from "ever scored" in the stats, the field name and logic will need refinement.
- 3,340 unscored leads remain in the pool. At ~3% warm conversion, that's ~100 more warm leads available with another backfill run via `onlyUnscored=true`. Roughly $10-15 to complete. Not urgent.

## Session 10 backlog (parked for session 11+)

- Continue scoring backfill with `onlyUnscored=true` to drain the remaining 3,340 unscored leads (~$10-15, ~100 expected warm conversions)
- Watch the agent live: how often does Don approve? Reject? What template performance shows after a week of real activity?
- Hot tier templates (score >= 40) - still no templates for hot leads. Currently they fall through to warm templates.
- iMessage seller variants - still no seller templates on the iMessage channel
- Cooldown re-touch templates - second-attempt copy after rejected drafts
- More buyer/seller template variants - only `viewed_listing_recent` has variants (sessions 9 fix)
- FUB UI custom fields 157-162 visibility to admins-only - carryover from session 4
- Normalize hideIfEmpty across fields 157 vs 160/161/162 - cosmetic
- Stale tag cleanup on person 27764 (`agent_draft_email_` trailing underscore)
- Drop body column after deprecation window
- Multi-agent: Ashley / Brenda / Taylor + unified Don queue
- v2: tier-based behavior, smart channel pick, production inbound classifier

## Session 9 — 2026-05-11 (buyer template diversity)

### What shipped

Two new buyer template rows + one selectTemplate code change. The selector previously did .find() on an exact scenario string match, so adding .vN variants would have been invisible to the cron. Patched to .filter() against a regex that matches the base scenario OR any .vN suffix, then random-pick when multiple variants match.

- **Template 15** `v1.warm.viewed_listing_recent.email.v2` opens with the property. Subject "Still circling {{address_or_area}}?"
- **Template 16** `v1.warm.viewed_listing_recent.email.v3` assumes warm and asks for the showing. Subject "Want to see {{address_or_area}} in person?"
- Template 1 (existing) untouched as v1 of the family.

All three share identical slot definitions so the slot-fill prompt requires no changes.

**Code change (commit 808c456)** in lib/agent/draftGenerator.ts selectTemplate function. The exact-string find() became a regex filter() that matches the base scenario OR `.vN` suffix, with random pick when multiple variants are eligible. Fully backward-compatible: templates without a .vN suffix continue to work as the sole match.

### Verification

Ran cron with limit=9, 5 buyer matches landed: 2 v1 + 0 v2 + 3 v3. v2 not hitting in 5 rolls is plausible variance (probability ~5-15% on a single 5-roll trial). v3 hitting confirms the regex matches variants. All 3 templates verified active=true, channel=email, lead_type=buyer. Accepted as variance and moved on.

### Followup observation (not blocking)

Looking at the rendered drafts side-by-side, within-variant diversity is weak when leads lack signals data for slot fill. Three v3 drafts all defaulted to "that listing" for `address_or_area` and two to "today" for `recency_phrase`. Cross-variant diversity works (v1 reads obviously different from v3) but within a variant, similar-shaped leads produce similar-shaped output. To improve, could enrich slot fallbacks or add more LLM-fillable slots. Parked for future iteration; current state is shippable.

## Session 8 — 2026-05-11 (template paragraph length sweep)

### What shipped

Three seller-track email templates had paragraph 3 over the 250-char cap, baked into the template body before any slot fill. Found via a 1-query sweep across all active templates, fixed via three UPDATE statements through Supabase MCP. No code change required — pure data.

- **Template 9** `v1.warm.seller_report_engaged.seller` — para 3 was 252 chars. Dropped "actually" and "specific" as filler words. New para 3 = 235 chars.
- **Template 2** `v1.warm.long_dormant_buyer.email` — para 3 was 252 chars. Dropped "straight up" and "actually". New para 3 = 231 chars.
- **Template 11** `v1.warm.long_dormant_seller.seller` — para 3 was 286 chars (worst offender). Tightened "still thinking about a sale" → "still thinking about selling", "did life take you a different direction" → "did life take you elsewhere", dropped "actually" and "probably", removed "in the area". New para 3 = 222 chars.

After fixes, regenerated drafts for John Miller (23316), Jay Miller (24825), Rachel Delmore (24498) — all 3 inserted clean with no block_reason. Paragraph 3 lengths now sit at 235/235/175 chars respectively.

### Sweep query for future reference

This query catches any active template paragraph over the cap. Worth re-running before any new templates ship:

```sql
SELECT 
  t.id, t.scenario, t.channel, t.lead_type,
  para.ordinality as para_num,
  length(para.paragraph) as para_len,
  left(para.paragraph, 80) as preview,
  CASE 
    WHEN length(para.paragraph) > 250 THEN 'OVER CAP'
    WHEN length(para.paragraph) > 230 THEN 'tight - risky after slot fill'
    ELSE 'ok'
  END as status
FROM fub_agent_message_templates t
CROSS JOIN LATERAL unnest(string_to_array(t.body, E'\n\n')) WITH ORDINALITY AS para(paragraph, ordinality)
WHERE t.active = true
ORDER BY length(para.paragraph) DESC;
```

After session 8 fixes, the worst paragraph across all 14 active templates is 235 chars — comfortable 15-char headroom under the 250 cap. Worth flagging template 13 paragraph 2 at 223 chars and template 10 paragraph 2 at 222 chars as "tight" — those have less than ~30 chars of headroom for slot fills like `{{last_outreach_phrase}}`. If those start blocking, the same trim approach applies.

### Pattern recognition for future template work

All three offending paragraphs followed the same structure: two complete thoughts joined into one paragraph by a sentence boundary. The trim path (drop 2-4 filler words) preserves voice better than splitting into 2 paragraphs, because splitting would push the template over the 4-paragraph max. **Rule of thumb for template authors:** if a paragraph has both an offer and a clarification, those usually want to be two adjacent sentences with light filler-word density rather than a single dense paragraph.

## Session 8 backlog (parked for session 9)

### Issue 2: Buyer template `v1.warm.viewed_listing_recent.email` produces voice-clone drafts

[Carried over from session 7 backlog, no change in scope. 5 buyer drafts in queue right now (14, 16, 18, 20) are all near-identical content. Lean toward path (b): 3-4 phrasing variants of viewed_listing_recent with random pick in selectTemplate.]

### Issue 3: hideIfEmpty inconsistency on Para fields

[Carried over.]

### Decide agent_enabled flip strategy

[Carried over. Templates are now clean, code is solid, real outbound verified. Probably ready to flip true with daily_send_cap=10 for week 1 once buyer template diversity lands.]

## Session 7 wrap-up — 2026-05-11

### What shipped

**Architecture redesign complete.** Email body now split across 4 FUB custom fields (Body=para1 + Para2 + Para3 + Para4, IDs 157/160/161/162) with newline-based paragraph stitching in Template 1156. The platform constraints discovered in Session 6 (256-char silent truncation, API-side HTML escaping) are no longer load-bearing.

**Phases completed in order:**
1. Audit + schema migration: paragraphs text[] column on fub_agent_message_drafts with CHECK constraint via fub_agent_paragraphs_valid() IMMUTABLE function, NOT VALID so historical rows are exempt. body column now NULLABLE.
2. Reverted commit 6a1fa72 cleanly (revert commit 76ba8a2). lib/agent/emailFormat.ts and scripts/backfill-html-email-bodies.ts deleted.
3. Hard-deleted 3 pending drafts (Gerard #7, Seanna #8, Anthony #10) that had <br><br> contamination. Jay #9 migrated in place as audit row, status='approved' preserved.
4. Created 3 new FUB custom fields via POST /v1/customFields: id 160 (Para2), 161 (Para3), 162 (Para4). Note: these were created with hideIfEmpty=false vs. field 157's hideIfEmpty=true. Cosmetic only; can normalize later.
5. Code refactor (commit 504f116): 9 files. lib/agent/fubAgentConstants.ts adds Para2-4 + FUB_CF_AGENT_DRAFT_EMAIL_BODY_FIELDS tuple + per-paragraph cap. lib/agent/paragraphs.ts NEW (splitBodyToParagraphs, validateParagraphs, padParagraphsToFubFields). draftGenerator.ts splits rendered body, validates against 250-char cap, inserts with block_reason on failure. fubPusher.ts writes 4 fields with empty-string padding so stale paragraphs from prior longer drafts get cleared on FUB person record. Edit route re-splits + re-validates, clears block_reason on success. Queue UI renders paragraphs individually with per-paragraph char counts, amber warning banner + disabled Approve when block_reason=paragraph_too_long. types.ts updated for nullable body + paragraphs field. log.ts added draft_created_with_block + fub_push_blocked event types.
6. Template 1156 update done via FUB UI: 4 merge tokens (%custom_agent_draft_email_body% + Para2 + Para3 + Para4) separated by actual newlines typed via Enter. Critical correction from spec: <br><br> separators do NOT work because FUB's Automation 2.0 Send Email step delivers as plaintext (or mail client picks plaintext alternative), so <br> arrives as literal text. Newlines work in both plaintext and HTML modes.
7. Pre-flight test against FUB person 27764 (Brian-in-Sphere) confirmed all 4 paragraphs render with breaks, no truncation, no escaped HTML.
8. Production verification: generated draft 17 for Jesse Hernandez (FUB 11323) via cron path, manually approved via queue UI, email landed in Jesse's inbox via Automation 2.0. Audit chain clean: lead_classified → draft_created → gate_check → draft_approved → fub_push_success in 532ms.

**Constraint-violation bug found and patched same-session (commit c8f0d51).** During Phase 7 verification, regenerated drafts for John Miller 23316 and Jay Miller 24825 failed at DB insert with "violates check constraint paragraphs_length_check". Root cause: draftGenerator.ts always inserted the paragraphs array even when validateParagraphs returned not-ok. The CHECK rejected the row and the lead disappeared from anywhere visible. Fix: insert paragraphs=null when blockReason is set. block_reason still populated for queue UI surfacing.

### Learnings captured in session 7

1. **FUB Automation 2.0 Send Email step delivers as plaintext (or the plaintext-alternative wins), NOT HTML.** The session 7 spec assumed `<br><br>` between template merge tokens would render as line breaks. In practice the recipient sees literal `<br><br>` text. Real newlines (typed via Enter in the FUB template editor) work in both plaintext and HTML modes. This is a permanent platform truth: never put HTML in FUB email templates for the agent.
2. **FUB UI Merge Fields dropdown produces `%snake_case%` syntax, not the camelCase API field names.** Field 157 API name is `customAgentDraftEmailBody`, but the merge token in templates is `%custom_agent_draft_email_body%`. Don't try to type merge tokens — use the dropdown.
3. **Postgres CHECK constraints cannot contain subqueries directly, but CAN call IMMUTABLE functions that contain subqueries.** First migration attempt with `SELECT max(length(p)) FROM unnest(paragraphs)` in the CHECK clause failed with `cannot use subquery in check constraint`. Wrapping in `CREATE FUNCTION ... LANGUAGE sql IMMUTABLE` then using the function in the CHECK works. Saved as `fub_agent_paragraphs_valid(text[])`.
4. **NOT VALID is the right tool for adding constraints when historical rows might violate them.** `ADD CONSTRAINT ... CHECK (...) NOT VALID` enforces all future INSERT/UPDATE without scanning existing rows. Used for paragraphs_length_check because Jay's audit draft (status=approved, body=423 chars from CP4) would have been rejected by a regular CHECK.
5. **The kill switch (agent_enabled=false) blocks BOTH cron AND manual approve.** Earlier session work treated agent_enabled as cron-only, but the outboundGate checks it on every send. To do verification sends with agent_enabled=false, flip it true momentarily, do the send, flip back to false. Better design than what I initially assumed; correct safety behavior.
6. **The generator's structured paragraph output already existed.** Templates have `\n\n` paragraph separators baked into their body strings. splitBodyToParagraphs(rendered.body) just splits on those. No prompt rewriting needed; the generator was always producing paragraph-structured content, we just weren't storing it as an array.

## Session 8 backlog (parked for next session)

### Issue 1: Seller template `v1.warm.seller_report_engaged.seller` paragraph 3 is 252 chars (over the 250 cap, before slot-fill)

3 of 3 seller drafts attempted with this template have blocked on paragraph_too_long: John Miller 23316 (draft 19), Jay Miller 24825 (draft 21), Tamela Karnazes 13307 (draft 22). Same paragraph, same overflow, every time.

The offending paragraph is hardcoded template prose:

```
The number on the report is a solid starting point but it doesn't know what your home actually shows like or what's trading on your specific block. If you're getting closer to listing, or just want a real PMV before you decide, happy to walk through it.
```

That's 252 chars literal. Slots in this template (`{{first_name}}`, `{{recency_phrase}}`) don't even land in this paragraph — it's the template body itself that's too long.

**Fix:** edit the template body in the DB (fub_agent_message_templates row where scenario = 'v1.warm.seller_report_engaged.seller'). Either split into two paragraphs on the period after "specific block" OR trim ~10 chars of filler. Same fix likely applies across other seller templates that haven't been exercised yet; sweep all seller templates and validate each paragraph against 250 chars at template-load time.

### Issue 2: Buyer template `v1.warm.viewed_listing_recent.email` is producing voice-clone drafts

5 buyer drafts generated this session (Gerard, Seanna, Anthony, Sam, plus Gerard regen): same template, same paragraphs, near-identical content. Only variables are first_name and a 1-3 word time phrase ("recently" / "a few days ago" / "last week"). Reads like a mail-merge if anyone receives multiple agent emails over a few weeks.

**Root cause:** the template has 1-2 LLM-fillable slots and a large amount of hardcoded text. The LLM's slot fill is essentially Mad Libs.

**Fix options for session 8:**
- (a) Expand slot count in buyer templates so more of the body is LLM-generated (more variation, but harder to keep voice consistent)
- (b) Create 3-4 buyer template variants in the `v1.warm.viewed_listing_recent` family so selectTemplate's priority cascade can rotate them randomly per draft
- (c) Just accept the cloning since daily_send_cap=10 limits exposure across recipients — unlikely two recipients compare emails

Recommend (b) — write 3 alternate phrasings of the same scenario, store as separate scenarios, and add a random pick step in selectTemplate when multiple scenarios match.

### Issue 3: hideIfEmpty inconsistency on Para fields

Field 157 (Body) has hideIfEmpty=true. Fields 160/161/162 (Para2/3/4) have hideIfEmpty=false. Means empty Para2-4 will show as empty fields in the FUB person profile UI, where Body hides itself when empty. Cosmetic; not blocking. Normalize via PATCH /v1/customFields/{id} if it bothers you.

### Test rig housekeeping

- Agent Test - Brian (FUB ID 31924) was deleted during session 7. Test rig now uses FUB ID 27764 (real Brian person, in Sphere stage, pond 9 which is POND_BRIAN_EXCLUDED so never picked up by agent). Keep this as the test rig going forward.
- Person 27764 has a stale tag `agent_draft_email_` (trailing underscore, no "ready" suffix). Probably from earlier session work. Safe to leave or clean up.

### Queue state at session 7 end (9 drafts in flight)

- Drafts 14, 16, 18, 20: buyer template `viewed_listing_recent`, clean, ready to approve. Same template; pick at most one or two if approving. (Or wait for session 8 template diversity fix.)
- Drafts 19, 21, 22: seller template `seller_report_engaged`, blocked on paragraph_too_long. Surfaced in queue with amber warning + disabled Approve. Need session 8 template fix OR manual edit.
- Draft 23: seller template `listing_thinking`, clean. Jesse Hernandez, same lead that received the verification send. Skip (probably reject) since he just got the other email.
- Draft 17: Jesse's approved+sent verification draft. Real outbound. Audit row.

## Session 7 entry — 2026-05-11 (smoke test attempted, CP4 failed)

[Original session 7 kickoff entry preserved below — describes the CP4 failure that motivated this whole session]

### What happened

Ran the 4-checkpoint smoke test gate end-to-end. Three of four passed cleanly. CP4 (first real outbound to Jay Miller, draft id 9) revealed TWO foundational FUB platform constraints that the current architecture doesn't handle:

1. **FUB custom fields silently truncate at ~255 characters on API write.** Jay's body was 423 chars; FUB stored only 255 of them, cut off mid-word ("doesn't know wha"). The POST returned 200, no error, our pusher logged `fub_push_success`, and Viktor's Automation 2.0 fired and delivered the truncated email to Jay's inbox.
2. **FUB API HTML-escapes text custom field values on write.** Our `<br><br>` got stored as `&lt;br&gt;&lt;br&gt;` in field 157, then rendered as literal `<br><br>` text in the delivered email. This is opposite behavior from when HTML is typed into a custom field via FUB's UI HTML editor — UI writes preserve HTML, API writes escape it.

Jay received a confusing email at 10:56 AM ET (14:56 UTC). Brian sent a personal recovery email from his own account within 10 minutes. Damage contained.

### Diagnostic confirmation via FUB MCP (read after CP4 failure)

Pulled Jay's record via `getPerson(24825)` to see what FUB actually stored:

- `customAgentDraftEmailBody`: 255 chars, ends mid-word ("doesn't know wha"). The `<br><br>` we sent is stored as `&lt;br&gt;&lt;br&gt;`. Both problems confirmed.
- `customAgentDraftEmailSubject`: "More than a number on the report?" — stored correctly (under 256, no HTML).
- Tags: `agent_email_sent` (Automation 2.0 step 2 fired), plus the lead's existing tags. `agent_draft_email_ready` was already removed by the Automation's tag-removal step.

## Session 6 wrap-up — 2026-05-09 (PM status reconciliation)

Brian confirmed Viktor's Automation 2.0 is fully wired and complete. The build is sitting at the smoke-test gate.

[Prior session 6 content preserved below this point - see session entries 1-6 history]

## Session 6 (cont.) — 2026-05-09 (PM stopping point)

What shipped:
- Pre-flight verification: GET `/v1/customFields` confirmed exact-name match for `customAgentDraftEmailBody` (id 157), `customAgentDraftEmailSubject` (id 158), `customAgentDraftIMessageBody` (id 159).
- Repo-root `CLAUDE.md` created with session-end checklist.
- `BRAIN_WRITE_TOKEN` moved to env var. `.env.local.example` shipped.
- `fub_cleanup/` decommissioned. Stale API key confirmed revoked.
- Brain push pattern proven end-to-end.

## Session 6 (cont.) — 2026-05-09 (AM micro-task)

FUB email template created via API. `POST /v1/templates` returned id 1156. Round-trip verified via GET. **NOTE: round-trip verification was insufficient — see session 7 learnings.**

## Session 6 — 2026-05-08

FUB email shell artifact at `scripts/fub-email-shell.md`. Critical gotcha (now superseded): the email step "must be plaintext, not HTML." This guidance was based on incorrect assumption about FUB's template editor having a plaintext mode — it does not. Email templates are HTML-only. (Session 7 update: the email is DELIVERED as plaintext regardless of how the template is constructed in the editor. The "plaintext" instinct was right; the implementation path was wrong.)

## Session 5 — 2026-05-07

Timezone helper, daily-cap flush cron, inbound classifier wired to side effects, cooldown reason enum extended, draftGenerator optout skip, smoke test SQL artifact.

## Session 4 — 2026-05-07

FUB pusher, reject taxonomy with 5-class enum, cooldown table, daily cap default 10, queue header with kill switch, inbound classifier stub, three FUB custom fields created (157/158/159), two FUB tags scheduled.

### Learnings captured in session 4

1. FUB tag list endpoint not accessible to standard API key.
2. Schema realism beats spec when adding new tables that reference existing ones.

## What this is

An AI-augmented lead nurture layer for HGPG. Scores Brian's eligible FUB leads using a hybrid rules + LLM intent engine, drafts personalized messages, and uses FUB Automations 2.0 (with Sendblue for iMessage) to do the actual sending. The agent does the *thinking* (who, what, when). FUB does the *sending*.

## Architecture (v2 - session 7 multi-field paragraph stitching)

- Embedded in TM repo (`HGPG1/hgpg-transaction-manager`) under `lib/agent/` and `app/agent/`
- Supabase: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel: TM project
- LLM: Anthropic Haiku 4.5 via `AGENT_ANTHROPIC_API_KEY`
- Auth: gated to `brian@homegrownpropertygroup.com`
- Cron: `/api/agent/cron` + `/api/cron/agent-daily-flush`
- Email body split across 4 FUB custom fields (160/161/162 added 2026-05-11) to dodge FUB's 256-char silent truncation. FUB Template 1156 stitches the 4 paragraphs with newlines (NOT `<br>` tags - those render as literal text in delivered email).

## Tables (8 fub_agent_* tables, RLS-locked to service role)

| Table | Purpose |
|---|---|
| `fub_agent_leads` | Denormalized FUB person snapshots |
| `fub_agent_lead_scores` | Score history |
| `fub_agent_message_templates` | Template library |
| `fub_agent_message_drafts` | Pending/approved/sent/discarded/blocked drafts. paragraphs text[] added 2026-05-11; body NULLABLE. |
| `fub_agent_lead_optouts` | Source of truth for opt-outs |
| `fub_agent_lead_exclusions` | Hard exclusions |
| `fub_agent_lead_cooldowns` | Per-lead temporary cooldown windows |
| `fub_agent_config` | KV thresholds + master kill switch |
| `fub_agent_log` | Full audit trail |

## Configuration as of session 7 end

| Key | Value |
|---|---|
| `agent_enabled` | `false` (flipped on briefly during session 7 Phase 7 verification, flipped back) |
| `auto_below_threshold` | `false` |
| `hot_threshold` | `40` |
| `warm_threshold` | `30` |
| `llm_confidence_floor` | `0.4` |
| `daily_send_cap` | `10` |
| `voice_file` | `brian-voice.md` |
| `scoring_version` | `v1` |

## FUB custom fields

| ID | API Name | Label | Purpose |
|---|---|---|---|
| 157 | customAgentDraftEmailBody | Agent Draft Email Body | Paragraph 1 (was full body pre-session-7) |
| 158 | customAgentDraftEmailSubject | Agent Draft Email Subject | Email subject |
| 159 | customAgentDraftIMessageBody | Agent Draft IMessage Body | iMessage body (single field, plaintext via LoopMessage) |
| 160 | customAgentDraftEmailPara2 | Agent Draft Email Para2 | Paragraph 2 (session 7) |
| 161 | customAgentDraftEmailPara3 | Agent Draft Email Para3 | Paragraph 3 (session 7) |
| 162 | customAgentDraftEmailPara4 | Agent Draft Email Para4 | Paragraph 4 (session 7) |

## Pilot scope

Brian-only v1. Buyer-only v1 (sellers in v2, though Jay was a seller draft test). Seller templates exist but session 7 surfaced that seller template `seller_report_engaged` has a too-long paragraph; awaiting session 8 fix.

## Open items going into session 8

- Fix seller template `v1.warm.seller_report_engaged.seller` paragraph 3 (252 chars hardcoded, blocks 100% of drafts using this template)
- Sweep all seller templates and validate each paragraph ≤ 250 chars at template-load time (consider a one-shot script that runs once over the template table)
- Add buyer template variation to break the voice-clone problem
- Decide whether to flip `agent_enabled=true` for sustained operation OR keep it manual-approve-only for week 1 ramp
- Scoring sweep on 4,340 unscored eligible leads (still deferred from session 6)
- Hot tier templates, iMessage seller variants, cooldown re-touch templates (still deferred)
- FUB UI visibility on custom fields 157-162 to admins-only (still carryover from session 4)
- Normalize hideIfEmpty across fields 157 vs 160/161/162 (cosmetic)
