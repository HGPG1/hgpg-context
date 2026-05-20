<!-- Last Updated: 2026-05-21 -->

# Session Handoff

## Last session: 2026-05-21 — /api/meta-insights date window bug fixed (Option A) + loud-error guard 🟢

### Context
Ads project handed over a spec (`META_INSIGHTS_DATE_WINDOW_FIX_SPEC.pdf`) flagging that 7d/30d/90d all returned identical totals ($3,253.79 / 295 leads). Same class as the Pipeboard scoping bug from 2026-05-19: the MCP wrapper silently ignores `date_preset` and falls through to its default `time_range="maximum"` (lifetime). Spec proposed Option A (swap `date_preset` for explicit `time_range: {since, until}`, ~10 min) and Option B (90d daily pull + server-side aggregation, ~1 hour) as fallback.

### What shipped
1. **Option A landed in `app/api/meta-insights/route.ts`** (commit `96b256a`). `buildScopedInsightsArgs()` now sends `time_range: { since: fmt(today-days), until: fmt(today) }` instead of `date_preset: DATE_PRESETS[days]`. `DATE_PRESETS` map kept as-is — still serves as the validation allowlist for the `days` query param.

2. **Verification (post Pipeboard Pro upgrade):**
    - 7d: $603.74 / 106 leads
    - 30d: $1,193.84 / 249 leads
    - 90d: $2,589.71 / 295 leads
    
    Monotonic across windows, 7d well under the old broken $3,253.79 lifetime figure. CPLs $5.70 / $4.79 / $8.78 — within expected range.

3. **Loud-error guard in `callTool()`** (commit `4bdf1b0`). Detects Pipeboard's `isError: true` envelope AND plain-text quota/plan-limit responses (matched via lowercase substring on "weekly limit", "trial commands", "upgrade to", "rate limit", "quota", or "error"-prefixed text that isn't JSON). Throws so the route's existing 502 path surfaces the actual error message. Prevents the silent-`$0`-spend regression that masked today's rate-limit hit for ~30 min.

4. **`projects/transaction-manager.md`** — added `date_preset` ignored bullet to the Pipeboard MCP gotchas section, alongside the existing scoping-args and filtering-array gotchas. Commit `2c0b60f`.

### The rate-limit detour (worth remembering)
First Option A verification returned `$0.00 / 0 leads` across all three windows. Spent ~20 min chasing potential bugs in the date math, the JSON-RPC payload shape, and Meta's `today`-vs-`yesterday` exclusion quirk before Brian flagged that his Pipeboard Free-plan dashboard read **100/30 weekly executions** — the previous session's debugging had blown through the quota. Pipeboard's quota error came back as a plain-text content block (not the JSON-RPC `error` envelope `pipeboardCall()` already catches), so `extractRows()` swallowed it as zero rows. Upgraded to Pipeboard Pro ($29.90/mo, 500 weekly executions, 7-day trial), re-ran the curls, Option A confirmed working. Loud-error fix shipped immediately to prevent recurrence.

### Brain writes this session
- `app/api/meta-insights/route.ts` — Option A (date_preset → time_range) + loud-error guard in callTool
- `projects/transaction-manager.md` — Pipeboard MCP gotchas updated
- `SESSION-HANDOFF.md` — this entry

### Lessons / patterns
- **Pipeboard's MCP wrapper ignores params silently AND inconsistently.** Already documented: `filtering` array (ignored), scoping args on `get_campaigns/get_adsets/get_ads` (ignored). Now add: `date_preset` (ignored, falls through to `time_range="maximum"` default). The pattern: when Pipeboard ignores a param, it returns lifetime/unscoped data rather than erroring. Belt-and-suspenders all date/scoping params until proven honored.
- **Pipeboard's `get_insights` tool schema accepts `time_range` as EITHER a preset string OR a `{since, until}` object** — both documented and both forwarded to the Graph API. The object shape is what works for arbitrary windows.
- **Plain-text error responses bypassed JSON-RPC error handling.** `pipeboardCall()` already throws on `parsed.error`, but Pipeboard's plan-limit message arrives as a successful JSON-RPC result with the error text in `content[0].text`. Always check for `isError: true` on result envelopes AND apply a heuristic plain-text error sniff for cases where the upstream doesn't set the flag.
- **Default-to-`maximum` is the silent failure mode of date_preset.** The 2026-05-19 dashboard build was running on lifetime data the whole time and nobody caught it because the numbers looked plausibly "high enough." Always verify time-filtered API responses by sanity-checking that different windows return different totals — the cheapest test is the one in the spec PDF.

### Open / parked from this session
- **Option B (90d daily pull + server-side aggregation)** is no longer urgent (Option A works on Pro) but still worth shipping as polish: drops Pipeboard calls per dashboard load from ~4 to ~1, unlocks the parked trend-sparkline enhancement from 2026-05-19. ~1 hour. Not blocking anything.
- **`/api/debug-meta-ads` cleanup** still slated for removal per `projects/transaction-manager.md` line 137.

### Pickup for next session
- Ads project can now re-run the 7d/30d performance review against correct date windows — ping queued in this Tech & Builds session response.
- If `/api/meta-insights` ever returns a 502 with `Pipeboard tool error (...)` in the message, the loud-error guard is doing its job — the upstream is the problem, not the route. Pipeboard's response copy is the diagnostic.
- Pipeboard Pro trial is 7 days. Mark calendar around 2026-05-28 to decide on continuing.

---

## Earlier session: 2026-05-20 (PM) — /api/meta-insights bearer-token auth shipped 🟢

### Context
Ads project handed over a spec (`META_INSIGHTS_BEARER_AUTH_SPEC.pdf`) to add bearer-token auth to TM's `/api/meta-insights` so Claude sessions in the Ads project can query the dashboard's enriched insights JSON directly, instead of Brian copy-pasting between sessions. Spec was tight: one-file edit at `app/api/meta-insights/route.ts` lines 461-465, plus a new `META_INSIGHTS_TOKEN` env var with Sensitive=OFF.

### What shipped
1. **`app/api/meta-insights/route.ts` patched** (commit `b7404392`). Bearer check runs first — `Authorization: Bearer <META_INSIGHTS_TOKEN>` short-circuits to OK. Falls through to existing `getSession()` + `BRIAN_EMAIL` check otherwise. Browser users still gated as before.

2. **Middleware exemption added** (commit `1bd766c2`). First verify attempt returned `{"error":"Unauthorized"}` (capital U) — that's middleware-level 401, not the route handler's `{"error":"Not authorized"}`. Middleware's `PUBLIC_PATHS` only lets bearer auth pass through for `/api/internal/*` or via `CRON_SECRET`; without an exemption, our route-level bearer check never ran. Added `/api/meta-insights` to `PUBLIC_PATHS` with comment "Bearer-auth or session check handled in-route". Same pattern as the `/api/internal/` exemption.

3. **`META_INSIGHTS_TOKEN` env var added** (Brian, Vercel dashboard, Sensitive=OFF, all three environments). Token: held in the Ads project's instructions, not pasted into the brain.

4. **Verification passed.** `curl -H 'Authorization: Bearer ...' '/api/meta-insights?days=7&level=campaign'` returned full enriched JSON: `date_range`, KPI totals (spend $3,252.16, 295 leads, CPL $11.02 over the 7-day window), campaigns array with status enrichment.

### Brain writes this session
- `app/api/meta-insights/route.ts` — bearer auth path
- `middleware.ts` — `PUBLIC_PATHS` exemption
- `projects/transaction-manager.md` — Meta Ads Dashboard section, new bearer-auth bullet
- `SESSION-HANDOFF.md` — this entry

### Lessons / patterns
- **Spec accuracy doesn't equal completeness.** The Ads-project spec correctly identified the route file and the line range, but the spec author was looking at the route in isolation and didn't account for the middleware layer in front of it. When patching auth on a route inside a Next.js app, always check the middleware too. The route-level `{"error":"Not authorized"}` vs middleware-level `{"error":"Unauthorized"}` casing difference is the diagnostic tell — if the error message doesn't match what the route handler returns, you're being intercepted upstream.
- **Middleware `PUBLIC_PATHS` is the right exemption mechanism** for routes that handle their own bearer auth. Mirrors `/api/internal/` and `/api/imessage/inbound`. Don't try to make middleware bearer-aware of every new endpoint — let route handlers own their auth.

### Open / parked from this session
- None. All within-scope items resolved.

### Pickup for next session
- Ads project can now query TM's insights API directly. First Ads-session smoke test will confirm end-to-end.
- If future endpoints need similar bearer-auth treatment, the pattern is: (a) bearer short-circuit before `getSession()` in the route, (b) add to middleware `PUBLIC_PATHS`, (c) env var with Sensitive=OFF.

---

## Earlier session: 2026-05-20 (AM) — Supabase security advisor cleanup (HGPG Core) 🟢

### Context
Supabase emailed Brian with 2 ERROR-level security findings on HGPG Core (ioypqogunwsoucgsnmla):
1. View `public.farm_campaign_roi` defined with SECURITY DEFINER (bypasses RLS on base tables)
2. Table `public._cma_packet_debug` is in public schema with RLS disabled

While in the advisor, also worked through the ~38 WARN/INFO-level findings.

### What shipped
1. **`farm_campaign_roi` view recreated with `security_invoker=true`** — same SELECT logic, now respects RLS on base tables (`farm_neighborhoods`, `farm_campaigns`, `farm_leads`). Service role still works (it bypasses RLS). End-user access would depend on per-table policies, but farm system isn't wired into any UI yet so this is theoretical.

2. **`_cma_packet_debug` drop-and-restore incident** — Initial read of the table was wrong. The underscore prefix and zero rows suggested old GLA-debugging scaffolding, so dropped it. But commit `b1c435d` from 2026-05-16 (4 days prior) had added live diagnostic logging to this table inside the CMA packet persist route, to capture Supabase error fields when Vercel log UI was truncating them. Dropping it tanked CMA packet generation entirely — every error path threw on a missing table. **Restored within minutes** with same schema, RLS enabled, anon/authenticated grants revoked, only service_role can write. Original security issue resolved; debug logging functional again.

3. **24 RLS-enabled-no-policy tables — anon/authenticated grants revoked.** These tables were "safe by RLS deny-all" but had full anon/authenticated SELECT/INSERT/UPDATE/DELETE grants. One bad migration adding a permissive policy or disabling RLS would have opened them up. Now they're properly locked down. Service role (used by all app code) unaffected.
    
    Tables locked: `amendment_applications`, `bg_activities`, `bg_agents`, `bg_contacts`, `bg_quiz_results`, `cma_feedback`, `concierge_notifications`, `farm_campaigns`, `farm_leads`, `farm_neighborhoods`, `fub_agent_config`, `fub_agent_lead_cooldowns`, `fub_agent_lead_exclusions`, `fub_agent_lead_optouts`, `fub_agent_lead_scores`, `fub_agent_leads`, `fub_agent_log`, `fub_agent_message_drafts`, `fub_agent_message_templates`, `internal_secrets`, `potentially_missed_emails`, `team_vendors`, `transaction_email_match_skipped`, `transaction_emails`.
    
    Highest-stakes one was `internal_secrets` which holds `DM_BRIAN_SECRET` (LoopMessage bearer for the dm-brian endpoint).

4. **5 trigger functions pinned to `search_path = public, pg_temp`** — `fub_agent_paragraphs_valid`, `set_agent_profiles_updated_at`, `bg_set_updated_at`, `farm_set_updated_at`, `parties_directory_touch_updated_at`. All trivial `updated_at` triggers, search_path hardening is mechanical.

### Advisor state after this session
- 2 ERRORs (the emailed ones): **gone**
- 5 function search_path WARNs: **gone**
- ~25 `rls_enabled_no_policy` INFOs: still present, but advisor downgraded to INFO since anon/authenticated grants are gone. These are noise now, not security issues.

### Remaining issues, intentionally deferred
**Cluster C — allow-all RLS policies on transaction_* and template tables (17 WARNs).** Tables: `transactions`, `transaction_milestones`, `transaction_parties`, `transaction_tasks`, `transaction_terms`, `transaction_documents`, `transaction_activity`, `transaction_deadlines`, `milestone_templates`, `milestone_audit_log`, `cma_reports`, `cma_adjustment_defaults`, `document_templates`, `task_templates`, `text_templates`, `reminder_templates`, `rezen_review_candidates`. These have `USING (true)` / `WITH CHECK (true)` policies that effectively bypass RLS. Currently fine because the apps all use service role and don't expose these via anon/authenticated PostgREST, but the policies should be tightened in a dedicated TM hardening session. Multi-app blast radius (TM, CMA Engine, tc-concierge) — don't do this hastily.

**Cluster D — `get_tracker_by_token` SECURITY DEFINER callable by anon (2 WARNs).** Intentional and correct. This is the public client tracker endpoint that lets buyers/sellers check transaction status via a token URL. SECURITY DEFINER bypasses RLS on `transactions`/`transaction_milestones`/`transaction_parties` to return token-scoped data. Leave as-is.

**Cluster E — `pg_trgm` extension in public schema (1 WARN).** Investigated 2026-05-20: pg_trgm has 3 active GIN trigram indexes on `transaction_emails` (property_address, snippet, subject — TM email matching) plus 10+ functions (`similarity()`, `word_similarity()`, `%>` operator, etc.). Moving extension out of public requires: (1) audit every repo for bare `similarity(` / `word_similarity(` / `%>` calls; (2) create `extensions` schema; (3) `DROP EXTENSION pg_trgm CASCADE` (kills the 3 indexes); (4) recreate extension with `WITH SCHEMA extensions`; (5) recreate the 3 indexes with `extensions.gin_trgm_ops`; (6) update Supabase connection search_path to `public, extensions`. Actual security risk is near-zero — only `postgres` and `service_role` have CREATE on public in Supabase. **Decision: deferred indefinitely as known-noise low-value lint.** Cosmetic Postgres hygiene flagged by Supabase's stricter project template; pg_trgm ships in public by default on every new Supabase project.

**Cluster F — Leaked password protection** ✅ Enabled by Brian 2026-05-20. HaveIBeenPwned check now active on signup/password-change in HGPG Core auth.

### Lessons / patterns
- **Don't drop scratch-looking tables without grepping the repo first.** Lock them down instead. The `_cma_packet_debug` incident wasted ~15 minutes and broke CMA generation. Underscore-prefix + zero rows + anon-writable looked like dead scaffolding but was live diagnostic logging from 4 days prior. Future heuristic: if uncertain, prefer "REVOKE + enable RLS + service-role grants" over "DROP".
- **`/api/external/log?repo=X` is your friend** for understanding why a DB object exists. Pulled the hgpg-cma-tool commit log and spotted `b1c435d` writing to the table.
- **Advisor INFO vs WARN levels are weighted by whether anon/authenticated have grants.** Removing the grants doesn't add a policy but it does downgrade the lint, which is exactly what you want for service-role-only tables.
- **`SECURITY DEFINER` functions are not all bad.** `get_tracker_by_token` is the canonical "intentional anon-callable SECURITY DEFINER" pattern: scoped by token, returns a precomputed JSON payload, no SQL injection surface. The advisor is paranoid; the pattern is correct.

### Open / parked from this session
- None blocking. All within-scope items resolved.

### Pickup for next session
- If a Supabase advisor email arrives again, only Cluster C remains as actionable. Cluster C (allow-all RLS policies on transaction_* tables) is a project, not a session. D (get_tracker_by_token) and E (pg_trgm) are intentional/cosmetic — leave as permanent advisor noise.
- Don't expect any user-visible regressions from this session's revokes. If TM, CMA, or any FUB-Agent surface fails with permission errors, it means a route is mistakenly hitting Supabase with the anon key instead of service role — that's the actual bug, not the revoke.

---

## Earlier session: 2026-05-19 (PM-2) — South Charlotte Report audit, brain corrected from repo ground truth 🟢


### Context
Brian asked for an audit of the South Charlotte Report pipeline + simplification suggestions. First pass was done without repo access (didn't realize `/api/external/read` existed on brain-app). Brian flagged the gap, second pass pulled repo ground truth via `/api/external/read?repo=south-charlotte-report` and corrected significant brain errors.

### What was wrong in the brain (now fixed)
- **Data source.** Brain said "MLS data fetch from Canopy MLS Grid (same source as CMA Engine)." Actually **RSS feeds + S3-scraped government/social sites** (from `south-charlotte-scraper`). Zero MLS involvement in the daily pipeline.
- **Email platform.** Brain project file said "via Resend"; `marketing.md` said "Beehiiv". Actually **Gmail SMTP** (`GMAIL_SENDER_EMAIL` + `GMAIL_APP_PASSWORD`).
- **Cron timing.** Documented as "7 AM ET digest send"; actually `0 10 * * *` UTC = **5 AM EST / 6 AM EDT**. Workflow comment in the repo itself is also wrong (says "6:00 AM EST" — should be "5:00 AM EST / 6:00 AM EDT"). Minor doc bug worth fixing next time touching repo.
- **Distribution.** `marketing.md` said "Instagram + YouTube" for the daily pipeline. Actually **WordPress + Instagram only** for daily. YouTube is a SEPARATE monthly pipeline.
- **Pipeline count.** Brain documented one pipeline; there are **two**:
  - **Track 1 (daily news):** `morning_fetch.yml` → `run_morning_fetch_enhanced.py` (182KB orchestrator) → RSS+S3 → AI filter → HeyGen → IG+WP+Gmail digest
  - **Track 2 (monthly HGPG market video):** `hgpg_market_video.yml` → `generate_hgpg_video.py`, triggered by `south-charlotte-market-report` repo via `repository_dispatch: market-report-ready`, posts to HGPG YouTube only
- **Repo count.** Brain documented 2 repos (`south-charlotte-report` + `south-charlotte-scraper`). Actually **3**: add `south-charlotte-market-report` (generates monthly market PDF, triggers Track 2). Added to `infrastructure.md`.

### Brain writes this session
- `projects/south-charlotte-report.md` — full rewrite from repo ground truth (commit `7fafaca`)
- `marketing.md` — SCR section corrected (commit `c5891ee`)
- `infrastructure.md` — added `south-charlotte-market-report` to active repos line (commit `b506f18`)
- `SESSION-HANDOFF.md` — this entry (replaces an earlier uninformed entry from same session)

### Audit findings (revised given ground truth, full version in project file)

The original audit critique was directionally right but specifics were wrong because I was working from stale brain. Corrected findings:

1. **Monolithic script is 182KB.** Real surgery to refactor, not a YAML reshuffle. Failure handling (digest-anyway, red banner, HeyGen short-circuit) is a workaround for absent checkpoints, but it works.
2. **Two-repo split for daily IS justified** — scraper writes to S3, report consumes from S3. Right boundary. Don't consolidate. (Earlier audit suggested consolidating — wrong.)
3. **HeyGen short-circuit already exists.** Worst-case failure is "no Reel today, blog still ships" — partial decoupling is already done. Better than I initially claimed.
4. **SCR YouTube is dormant.** `YOUTUBE_REFRESH_TOKEN` env var present but unused per HANDOFF. Decide: remove or document dormancy.
5. **Digest critique partially wrong.** Daily digest is to Brian only — no subscribers. The "red banner reaches subscribers" risk I claimed doesn't exist. If Beehiiv newsletter exists, it's a separate thing outside the workflow.
6. **No traffic feedback loop is real.** No UTMs on blog/YT links.

### Proposed 3-job split (still the right shape, but lower urgency)
- `fetch_and_filter` (10:00 UTC, always runs) → RSS + S3 scraper data + brand-safety filter → artifacts
- `publish_text` (10:30 UTC, needs fetch) → WordPress + Gmail digest, independent of video
- `produce_and_publish_video` (10:30 UTC, needs fetch) → HeyGen + IG Reel, allowed to fail without blocking text

Honest assessment: pipeline works, Brian called it ops-mode. The refactor is code health, not a fix. Wait for the next breakage to be the trigger.

### Open / parked
- **Refactor go/no-go on the 3-job split** — Brian to decide. Not urgent.
- **Workflow comment bug fix** — 1-liner, next time touching repo.
- **YouTube SCR dormancy** — decide remove vs. document.
- **Beehiiv claim verification** — is there a separate Beehiiv newsletter, or was the brain entry aspirational? Flagged in project file and marketing.md.
- **UTMs on blog/YT links** — if traffic feedback is wanted.

### Pickup notes
- Memory NOT updated this session — at 30/30 cap, SCR specifics belong in brain per rule #11.
- **`/api/external/read` exists** on brain-app and can read any HGPG1 repo via `?repo=<repo>&path=<path>`. Worth remembering for future audits — this session almost shipped a wrong audit because I didn't try it.
- No code touched, no commits to `south-charlotte-report` repo. Brain-only writes.
- Last 5 commits on brain (this session): `7fafaca` SCR project rewrite, `c5891ee` marketing.md SCR section, `b506f18` infra market-report repo add, `05c320e` SESSION-HANDOFF earlier (now stale entry replaced), `4f9593f` SCR project earlier (now superseded).


---

## Previous session (today PM): 2026-05-19 (PM) — Ads project blocked on Pipeboard token gating, bearer-auth spec handed to Tech & Builds 🟡

### Context
Brian (in Ads project) asked Claude to run a Meta Ads performance review across the new TM /meta-ads dashboard. The /api/meta-insights endpoint is session-gated (cookie auth, brian@homegrownpropertygroup.com check), so Claude in a separate project session cannot hit it directly. Currently requires copy-paste of JSON from browser, which is fine for one-offs but blocks streamlined ad reviews.

### What is parked with Tech & Builds
**Spec:** Add bearer-token auth path to /api/meta-insights as an additive second auth mechanism (session gate stays). Same pattern as Brain App /api/external/*.
- **File to change:** app/api/meta-insights/route.ts (lines 461-464)
- **New env var:** META_INSIGHTS_TOKEN in Vercel (Production + Preview + Development, Sensitive=OFF)
- **Effort:** ~10 min, single file edit + env var
- **Spec PDF:** META_INSIGHTS_BEARER_AUTH_SPEC.pdf (delivered to Brian 2026-05-19 PM)
- **Risk:** Low. Bearer check is additive. Rollback = delete env var.

### After ship, do these in order
1. Verify with {"error":"Unauthorized"}
2. Bump  Recent Ships with the bearer auth note
3. Paste the token to Ads project so it goes into project instructions
4. Run the Meta Ads performance review that triggered this work

### Watch-outs
- Vercel "Sensitive" env var gotcha (learned 2026-05-19 AM): Sensitive=ON causes the var to come into runtime as an empty string with no error. Use Sensitive=OFF.
- Make sure the bearer check is short-circuited (no getSession() call when token matches) so API calls don't hit the auth DB on every request.

---

## Last session: 2026-05-19 — Variant E launch reconciled, Buyers Guide cleanup pass, brain-app /log endpoint shipped, IDXRE B2 verified 🟢

### What shipped

**Brain reconciliation (Variant E):**
- Variant E launched 2026-05-18 with full scoped package (1:1 + 4:5 + 9:16 creatives, IMG_5675, $10K+ closing-cost hook, "Learn More" CTA, `utm_content=variant-e-10k`).
- Brain entries fixed: `projects/charlotte-new-construction.md` (status 🟡→🟢, read schedule logged, Variant E in Done/shipped), `marketing.md` (new dedicated Variant E section with brief/creative/UTM/benchmark), `CONTEXT.md` (active campaigns line + Recently completed).

**Buyers Guide code cleanup (4 commits to charlotte-buyers-guide):**
- `11a0a4a4` — bump Last Updated header on `api/fub-lead.ts` 2026-05-12 → 2026-05-13
- `2b405da3` — bump Last Updated header on `api/exit-intent/submit.ts`
- `2e722609` — bump Last Updated header on `api/bonus/unlock.ts`
- `9c3168c7` — strip orphan `ADVISOR_MODE_GUIDE.md` reference from `api/_lib/agents.ts` (file never existed in repo)
- Retracted earlier false copy-bug claim on `/admin` page (it correctly renders `<h1>Admin</h1>` — I misread the minified bundle on first pass)

**Brain-app feature shipped:**
- `/api/external/log` endpoint live at `brain.homegrownpropertygroup.com` (commit `2b73da1a` in HGPG1/brain-app). Read-only git-log proxy for any HGPG1 repo, same auth/validation pattern as `/api/external/read`. Optional filters: `path`, `since`, `until`, `ref`, `per_page` (cap 100), `page`. POST or GET. Returns sha, short_sha, message, full_message, author_name, author_email, author_date. Closed the gap where Claude sessions needed `gh` from Brian's Mac for commit history.
- `operations.md` updated with full 5-endpoint surface documentation
- `projects/brain-app.md` updated with `/api/external/read` + `/api/external/log` sections, auth-model heading bumped to "all four endpoints"

**Buyers Guide brain reconciliation:**
- Full Session 3 commit-by-commit history captured via the new /log endpoint (8 commits since 2026-05-13)
- Surfaced **2 commits that hadn't been documented anywhere** before today:
  - `2af5340` (2026-05-19 06:20 ET) — Post-Session-3 polish: phone format, FUB tags, `/owner` route dropped from public allowlist
  - `58687cc` (2026-05-19 08:29 ET) — Fix FUB quiz tag keys to match canonical Quiz values. **Silent bug: 4 of 5 buyer-type tags + 1 of 4 lifestyle tags weren't being applied for ~7 days (Session 2 ship → 2026-05-19).** Any Quiz lead captured 2026-05-12 to 2026-05-19 missing buyer-type/lifestyle FUB tag.
- New brain file `projects/buyers-guide-advisor-notes.md` (commit `f380f02a`) — full 40-talking-point inventory pulled live from `AdvisorNote-C4olMM97.js`, organized by buyer-facing page (/quiz, /neighborhoods, /strategy, /checklist). 1 of 12 blocks approved this session ("Setting up the strategy conversation"). Review pass paused.
- `projects/buyers-guide.md` corrections: fixed Lesson #19 (agent allowlist baked in, but agent DATA fetched from Supabase via `/api/agent/by-slug` and `/api/agent/default` — earlier claim of "no /api/agents endpoint" was wrong). Added Lessons #22 (silent enum-map drift post-mortem) + #23 (intentional two-tier allowlist divergence between client and server).

**IDXRE B2 verification (no code shipped, all live API probes):**
- Brain `projects/idxre-2026-04.md` and `CONTEXT.md` updated with verification results (commits `3ea8114b`, `2dd51167`).
- KTS pause held — all 27 Hot leads frozen at 2026-05-17T17:28-31 UTC, no activity since. Strong indirect signal that the bulk-pause Automation 2.0 worked.
- All B1 tier counts match brain exactly: Hot=27, Warm=105, Cool=239, Cold=778, Pond 17 Dead=135.
- **B2 build verified live:** Templates 1166 + 1167 exist (created 2026-05-18 18:23 UTC). Automations 336 (Sellers) + 337 (Buyers) exist (created 2026-05-18 20:15-20:29 UTC).
- **Narrative drift found:** brain said Viktor was building B2 automations. Live data shows `createdById: 1 (Brian McCarron)` on both. Either Brian built them himself or Viktor was working under Brian's login.

### What's parked

**Buyers Guide:**
- AdvisorNote review pass: 1 of 12 blocks approved, 11 remaining. Pickup at "Seller's market notes" next. Source-of-truth file in repo is `src/data/advisor-content.ts`. Approval marker format: `✅ APPROVED YYYY-MM-DD` next to block heading. Rework marker: `🔴 REVIEW:` prefix on a line.
- Mobile smoke test of AdvisorMode (iPhone/iPad) — test plan in advisor-notes file. Entry URL pattern: `buyersguide.homegrownpropertygroup.com/advisor/<page>`. Persistent device-level localStorage flag. Tap "Exit advisor mode" to clear.
- Session 4 (optional): Interactive Map + Market Pulse + bonus PDFs. ~1.5 hrs.
- Manus app takedown after S4.
- ⚠️ **FUB quiz tag backfill potential** — leads captured 2026-05-12 → 2026-05-19 may be missing buyer-type / lifestyle tags. Bg_quiz_results.buyer_type and lifestyle_priority are intact in Supabase; FUB tags are what's missing. Worth a one-off backfill script if you want clean historical data.

**IDXRE B2 launch gates (UI-only, can't be done via API):**
1. Verify audience filters on Automations 336 + 337: `IDXRE-Cool OR IDXRE-Cold + IDXRE-Seller` (336); same + `IDXRE-Buyer` (337). Hot/Warm exclusion on both.
2. Date-stamp tag baked in (`IDXRE-2026-05-XX-B2`).
3. Final copy eyeball on subjects + bodies (templates 1166/1167 read live this session, body content captured in `projects/idxre-2026-04.md` verification section).
4. Flip automations from draft → live.
5. Pick launch day: Tue-Thu this week = today/tomorrow/Thursday.

**Variant E observation window:**
- Day 3-4 first read: 2026-05-21 → 2026-05-22 (Thu-Fri)
- Day 5-7 decision point: 2026-05-23 → 2026-05-25 against Variant C's $4.63 CPL benchmark
- Success bar: E within 1.5x C's CPL at day 3-4 (≤ ~$7 CPL)

**Open from earlier sessions today (preserved):**
- Pipeboard token blocker — `PIPEBOARD_API_TOKEN` env var still needs to be added to Vercel TM project before the Meta Ads dashboard becomes useful. The Meta Ads drill-down session today shipped UI; the data source is still blocked on the token.

### Pickup notes for next session

- **`/api/external/log` endpoint is live and bearer-protected.** Use it to fetch commit history for any HGPG1 repo when reconciling brain entries. Same token, same pattern as `/read`. Operations.md documents the full surface.
- **AdvisorNote review file** lives at `projects/buyers-guide-advisor-notes.md`. The 11 unreviewed blocks are listed in source order; "Seller's market notes" is next.
- **IDXRE B2 is unblocked from a build standpoint.** Only UI-side gates remain. Brian can flip the automations live whenever he's done with the audience-filter verification.
- **The Viktor narrative on IDXRE B2 needs reconciliation** — was Viktor actually involved in the B2 build, or was that a brain memory artifact from an earlier plan? Worth a one-line note in idxre-2026-04.md once clarified.
- **Phone Capture Rate Review** — run window opened 2026-05-19 (today), still untouched. Query plan in `projects/new-construction-phone-capture-rate-review.md`.
- **TC Concierge real-life extraction review** — Don's been running real deals; classifications/extractions need a quality check.

## Previous session: 2026-05-19 — Meta Ads dashboard drill-down shipped 🟢

### What shipped
- TM /meta-ads now supports campaign → ad set → ad drill-down navigation. Click any non-unknown row to drill in. Breadcrumb at top for navigation back up. Ad level is terminal.
- Hide unknown status toggle in the tab row. Defaults ON. Count chip shows when unknown rows exist (e.g. "Hide unknown (12)"). Unknown rows are deleted/archived objects with no current Pipeboard metadata — usually noise.
- API extended: `/api/meta-insights` takes optional `level` (campaign|adset|ad) + `parent_id` + `parent_kind`. Backwards-compatible (no params = campaign-level as before).
- Server-side 60s response cache keyed by `${level}:${parentKind}:${parentId}:${days}` to absorb rapid drill-down clicks without burning Pipeboard's 200/hour rate limit.
- Fail-soft metadata: if Pipeboard rejects the get_adsets / get_ads tool calls, status pills fall back to "unknown" but the insights table still renders.
- Two files edited, both deployed cleanly on `dpl_CyGSmEx4DLUC1FFNu3LaVjaeCoDv` (commit `334ba6c`):
  - `app/api/meta-insights/route.ts`
  - `app/meta-ads/MetaAdsDashboard.tsx`

### Earlier in the same session (mid-day 2026-05-19)
- Built and shipped the initial /meta-ads dashboard (campaign-level only at that point). See `projects/transaction-manager.md` Recent Ships > Meta Ads Dashboard.
- Hit the Vercel "Sensitive" env var gotcha: PIPEBOARD_API_TOKEN was marked Sensitive on creation, which causes the runtime to receive empty strings for the var. Fix: delete + re-add WITHOUT the Sensitive toggle. Diagnosed via a temporary `/api/debug-env` route (now removed). Memory updated with this gotcha for future env-var debugging.

### Follow-ups during the same session

**Unknown statuses at adset/ad drill-down (FIXED).** Initial deploy showed all rows as "UNKNOWN" status at ad set and ad levels. Root cause: Pipeboard's MCP tools (`get_adsets`, `get_ads`) silently ignore scoping args like `campaign_id` / `adset_id` — they return the first N unscoped objects. So lookups for the actual drilled-into IDs missed the cache and fell back to "unknown". Fix: switched to fetching account-wide metadata maps (one per level) cached for 5 min, look up by ID locally. Account is small enough that wide fetches are cheap.

**Drill-down showing wrong ads (FIXED).** After the metadata fix, drilling into an ad set showed ALL ads in the account (with correct statuses now). Root cause: Pipeboard's MCP insights tool also ignores the `filtering` array param. It was returning every ad in the account with activity in the date range. Fix: switched to scoping via `object_id` (mirrors Meta's native `/<adset_id>/insights` URL-path scoping pattern). Kept `filtering` array as belt-and-suspenders. Verified working: drilling into a real campaign now shows only the 8 ads in that campaign, with correct statuses, leads, CPL.

**Vercel "Sensitive" env var gotcha rediscovered.** During initial /meta-ads setup, PIPEBOARD_API_TOKEN was added with the Sensitive toggle ON (which is default in some Vercel UI flows). Sensitive vars come into the runtime as empty strings (length 0, no error). Diagnostic pattern: `/api/debug-env` route that reports `present`, `length`, `truthy` for the var. Fix: delete + re-add the var with Sensitive OFF. Added to memory.

### Diagnostic patterns worth keeping
- `/api/debug-env` (already removed): reports env var presence/length/truthy without leaking values. Re-add with same shape if any future env-var-related not-configured error.
- `/api/debug-meta-ads` (still in repo as of commit `36cdabb`, needs removal): tests Pipeboard tool list + tool-call variants to diagnose which scoping params actually work. Useful template for other MCP integrations where you need to figure out what the wrapper actually honors.

### Pickup notes for next session
- Drill-down assumes Pipeboard's `get_adsets` and `get_ads` tools accept Meta-style scoping args (`campaign_id`, `adset_id`). If those don't scope server-side, the function still works (we filter the metadata client-side via the map), but it'll pull more data than needed. If you see ad set views taking >10s, check Pipeboard tool docs and add explicit filtering arg.
- Insights `filtering` param uses Meta Graph syntax: `[{"field":"campaign.id","operator":"EQUAL","value":"123"}]`. Pipeboard should pass this through. If drill-down returns empty data, the filtering shape might need adjusting per Pipeboard's preferences — fall back to client-side filter as a safety net.
- The 60s response cache is per-lambda-instance. Cold starts get a fresh cache. For Brian's single-user pattern this is fine.

### Open for future enhancements (parked)
- Custom date range picker (currently only 7/14/30/90)
- Scheduled performance digests (weekly email to Brian with top-line numbers)
- Click-through to Meta Ads Manager from each row (open the actual campaign in Ads Manager UI for editing)
- Trend sparklines on KPI cards
- "Compare to previous period" toggle

## Earlier session: 2026-05-19 — Buyers Guide Session 3 verified shipped 🟢

### What happened
- Brian flagged that Buyers Guide Session 3 was complete, but the brain still listed S3 as queued/parked.
- Pulled live state from production (`buyersguide.homegrownpropertygroup.com`) to verify what actually shipped. Confirmed: production was last deployed today 2026-05-19 13:45 UTC (09:45 ET). Session 3 IS live and accepting real traffic.
- Inspected the deployed JS bundle (`index-DHp9PF4a.js` and `agentDetection-R8ThZYk3.js`) to enumerate what was actually built.

### What's live (verified by bundle inspection + API probe + DB query)
- **Agent slug routes**: `/brian`, `/brenda`, `/ashley`, `/taylor` — each parses slug, persists to `localStorage` as `homegrown_assigned_agent_slug`, propagates via `AssignedAgentProvider` context
- **Reserved-routes table** in `agentDetection.ts` keeps `quiz, calculator, rent-vs-buy, neighborhoods, strategy, checklist, bonuses, thank-you, advisor, admin, agent-dashboard, api` from being misclassified as agent slugs
- **`AgentProfile` component** renders assigned agent's info in-page
- **Advisor Mode**: `/advisor/<path>` entry, `homegrown_advisor_mode` localStorage flag, persistent banner "Advisor Mode · Talking points visible to you only" with "Exit advisor mode" button
- **`AdvisorNote` components** embedded throughout buyer-facing pages, only render when `isAdvisorMode === true`
- **`assignedAgentSlug` propagation** to `/api/fub-lead`, `/api/exit-intent/submit`, `/api/bonus/unlock`
- **Server-side resolution**: each handler looks up `bg_agents WHERE slug = ?`, writes `bg_contacts.assigned_agent_id`, calls FUB `assignLeadToAgent`, returns `{success: true, contact: {...}, assignedAgent: {id, slug}}`
- **No new server endpoints** — agent list is statically baked into the bundle (hardcoded `[brian, brenda, ashley, taylor]`). No `/api/agents` collection endpoint. Tradeoff documented as lesson 19 in `projects/buyers-guide.md`.

### Verified real-traffic activity
4 leads with named `assigned_agent_id` captured today between 10:10 and 12:36 UTC, hitting agent IDs 30001 (Brian), 30003 (Ashley) via slug resolution. Full table in `projects/buyers-guide.md` "Verified live activity" section.

### Brain updates pushed this session
- `projects/buyers-guide.md` — status flipped to `🟢 Sessions 1+2+3 complete`, full Session 3 plumbing section added, Sessions 3 listing moved from Pending → Recent build history, 3 new lessons appended (#19 baked-in agent list, #20 reserved routes table, #21 localStorage attribution model). Commit `470432e3`.
- `CONTEXT.md` — "Last updated" bumped to 2026-05-19; "Recently completed" gained Buyers Guide S3 line; "Parked" entry rewritten to reflect only S4 outstanding. Commit `bf7dac68`.

### What's still pending on Buyers Guide
- **S3 post-deploy verification** (now in `projects/buyers-guide.md` Pending): confirm `/admin` and `/agent-dashboard` reserved routes render their intended UI (or were left as placeholders for a future session). Confirm advisor mode banner displays correctly on iPad and iPhone. Confirm `AdvisorNote` copy is reviewed before any agent uses it in a buyer consult.
- **Session 4 (optional)**: Interactive map, Market Pulse, bonus PDFs (~1.5 hrs)
- **Manus app takedown**: after S4

### Pickup notes for next session
- One test row was written to `bg_contacts` during this verification session (`probe-discardme@example.invalid`, id 13) and cleaned up via direct Supabase DELETE. No residue.
- If Brian wants the brain to track Session 3 commit SHAs (vs just the high-level "shipped"), need access to `HGPG1/charlotte-buyers-guide` git log — the project file lists the S2 commits but not S3. Worth fetching via gh CLI next time we're touching the repo.
- The Pipeboard token blocker from the TM Meta Ads dashboard session is still outstanding — Brian needs to add `PIPEBOARD_API_TOKEN` to Vercel TM project env vars before that dashboard becomes useful.

## Earlier session (2): 2026-05-19 — Meta Ads dashboard added to TM 🟢

### What shipped
- New TM route `/meta-ads` — internal Meta Ads performance dashboard, replacing manual CSV exports from Ads Manager (Pipeboard token now lives server-side, never in browser)
- New TM API route `/api/meta-insights` — server-side Pipeboard JSON-RPC proxy. Reads `PIPEBOARD_API_TOKEN` from env, resolves Pipeboard tool names once per process lifetime (1hr TTL), fail-soft on the campaigns call (status pills fall back to "unknown" if Pipeboard rejects that tool but insights still renders)
- Brian-gated end-to-end: server component redirects non-Brian sessions, API route returns 403 to non-Brian. Defense in depth so the token never gets exercised by anyone else
- Sidebar nav links added (`app/layout.tsx` desktop, `components/MobileNav.tsx` mobile), gated on `user.email === brian@...` matching the `/agent` pattern
- All 5 files committed via brain commit API. Final deploy `dpl_4LxM3f61sWuhUS9pZ1F7jXy7Ui1T` READY at production

### Files
- `app/api/meta-insights/route.ts` (new)
- `app/meta-ads/page.tsx` (new, server component)
- `app/meta-ads/MetaAdsDashboard.tsx` (new, client component, Tailwind matching TM conventions)
- `app/layout.tsx` (edit: nav link)
- `components/MobileNav.tsx` (edit: nav link)

### Architecture notes
- The UI is a 1:1 port of the Ads-project prototype (`meta_ads_dashboard.html`) with the token panel stripped. CTR normalization, lead extraction from `actions[]`, status pill colors, sort logic, CSV export logic all lifted verbatim
- Pipeboard endpoint: `https://meta-ads.mcp.pipeboard.co/` (JSON-RPC 2.0, Bearer auth, 200/hour rate limit). Account `act_31445287`
- 4 date ranges: 7 / 14 / 30 / 90 days (mapped to `last_7d`/`last_14d`/`last_30d`/`last_90d` Pipeboard presets)
- 5s refresh throttle on the Refresh button. Tab switches always force-fetch (no throttle)
- Page renders nothing useful until `PIPEBOARD_API_TOKEN` env var is set on Vercel — error box will display `PIPEBOARD_API_TOKEN not configured on server`

### Pickup notes for next session
- **Blocking: `PIPEBOARD_API_TOKEN` not yet set on Vercel TM project.** Brian to add via Vercel dashboard (Production scope minimum; Preview if testing on branch deploys). Token format starts with `pk_`. Get from pipeboard.co/api-tokens. Once set, the page works immediately on next request (tool name cache is per-process, so cold-start picks it up).
- One intermediate deploy errored (`dpl_8trNCqM7m2Yc4mHYLJPqgayeqCtz`) because `page.tsx` committed before `MetaAdsDashboard.tsx`. The 5 commits were sequential single-file calls so intermediate states existed briefly. Subsequent deploy fixed it. Not a problem in practice but a reason to bundle related files into a single push when possible (or commit them in dependency order).
- Future enhancements parked: ad set / ad creative drilldown, custom date range picker, scheduled performance digests (e.g. weekly email to Brian), multi-account support if Brian ever runs ads from a second account.
- Token rotation procedure if Pipeboard token expires: refresh at pipeboard.co/api-tokens, replace `PIPEBOARD_API_TOKEN` in Vercel, trigger a redeploy (or wait for the next push; tool-name cache invalidates per cold start).

## Earlier session (3): 2026-05-19 — Variant E scoping confirmed, creative render deferred 🟡

### What happened
- Brian asked whether Variant E (New Construction Incentives campaign) had been scoped here. It had — in an earlier same-day session.
- Confirmed Variant E is fully scoped and the brand-styled brief PDF is already shipped at Drive ID `1U6Voq92rqVSFDIrTFiborWCBKFbiLqk3` (`variant-e_brief.pdf`, created 2026-05-18 19:44 UTC, in Brian's My Drive root).
- Brief covers: dollar-anchor angle ($10K+ closing-cost hook), audience/placement table mirroring Variant C, 5 copy variants with headlines + descriptions, creative dimension specs (1080x1080, 1080x1350, 1080x1920), UTM table with `utm_content=variant-e-10k`, and the day-3/5/7 test plan against C and D.
- CTA in shipped brief is "Learn More" (not "Get Offer" — earlier draft in this thread proposed Get Offer but the locked brief uses Learn More; go with what shipped).

### What's still parked
- **Creative render is NOT done.** The brief references three PNG/JPG creatives (1:1, 4:5, 9:16) built around photo IMG_5675 (black polo, kitchen island, warm closed-mouth smile — same photo used in Variant D).
- IMG_5675 could not be located via Drive search this session. Folder ID `14G9ECCBAGgJJjau-IglmtDIOBPKlwouF` (51-photo lifestyle folder per brain memory) returned empty on multiple queries. Either the folder ID rotated, permissions changed, or the connector isn't indexing images in that folder.
- Three new photos uploaded to project this session (IMG_5681, IMG_5711, IMG_5720) — all Work Hard Be Kind shirt, disqualified per brand rule against competing personal branding on clothing.

### Pickup notes for next session
- **First move:** get IMG_5675 to Claude. Options that worked / didn't:
  - Drive search by name = empty
  - Drive search by parent folder ID = empty
  - Direct file ID would work via `Google Drive:download_file_content` if Brian pastes a link
  - Or Brian uploads IMG_5675 directly to the chat
- **Once photo is in hand:** render three creatives matching the locked brief spec — 1080x1080 square, 1080x1350 portrait, 1080x1920 vertical. Output both PNG (quality) and JPG (size fallback). Composition per brief: photo top with navy gradient, single hero number `$10,000+`, locality strip (Indian Land · Fort Mill · Waxhaw · Lake Wylie), HGPG + Real Broker lockup, CTA pill.
- **Audit before Brian launches:** confirm pixel ID `1880396459290092`, URL destination, UTM tags match brief table, Special Ad Category: Housing applied.
- **Day 3-4 first read; day 5-7 decision point** against Variant C's CPL. Success bar: E within 1.5x C's CPL at day 3-4.

### Audit note for marketing.md
- Variant E status in `marketing.md` should reflect: brief shipped, creative render pending, not yet launched. If the file currently lists Variant E as "launched" or "live," that's wrong — only the brief is done. Next session should reconcile.

### Drive connector note
- Multiple name-based and parent-based Drive searches returned empty this session. May be a connector indexing issue rather than missing files. Worth a smoke test next session — try `name contains 'IMG'` against a known-good folder to confirm the connector is actually searching image files.

---

## Last session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser → commit lands on `main` with author `brian@homegrownpropertygroup.com`

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration" (verify)
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS" (verify)
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation" (verify)
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Bugs found and fixed mid-session
- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback) — fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing — fixed permanently by switching to Resend custom SMTP

### Project status updates
- `projects/brain-app.md` — status now 🟢 SHIPPED (was 🟡)
- `projects/hgpg-team-tools2.md` — Site URL in Supabase still points here for the broken app's eventual fix
- `projects/transaction-manager.md` — no changes today, but TM benefits from Resend SMTP upgrade

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search

### Pickup notes for next session
- Brain-app is live and working — use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini — push from whichever machine you most recently ran `npm install` on

