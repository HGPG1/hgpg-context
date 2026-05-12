<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Narrative cache freshness bug closed

PR #43 (`polish-narrative-cache-fix`) — merged + deployed (READY, `dpl_BGQSLDhaghgjM37fBZcvKTEA3BhV`).

**The bug:** PR #42's caching effectively shipped **zero** savings. Brian re-tested Candlestick and the LLM was firing on every view. Verified directly on row `7cb7ef75`: `narrative_generated_at` 12:34:31 ET, then the PR #34 PDF-export status flip (draft → presented) bumped `updated_at` to 12:36:19 ET — 108 seconds later — without touching `narrative_generated_at`. By the time of this fix **all 47 narrative-bearing rows in HGPG Core were stale**.

**Root cause:** `cma_reports_set_updated_at` only bumped `narrative_generated_at` when the narrative column itself changed. Several legitimate UPDATEs bumped `updated_at` without touching narrative:
- PR #34 PDF-export status flip from the packet page
- Manual status changes from /history (presented → archived, etc.)
- Any future label edit (agent_name, subject_address)

Each landed `narrative_generated_at < updated_at` on the row → `loadCachedNarrative` correctly marked it stale → regen on every reopen.

**The fix (trigger, not call-sites):** per Brian's stop-and-ask, fixed the trigger as a single point rather than every UPDATE call site. New rules in `cma_reports_set_updated_at`:

| Update kind | `narrative_generated_at` |
|---|---|
| Math changed (`subject_summary`/`comps`/`adjustments`/`pmv`/`dom`/`pqs`) | leave alone → stale signal (intentional) |
| Narrative changed | stamp fresh |
| Neither changed, row already has narrative (status flip, label edit) | bump along with `updated_at` → cache stays valid |

`BEFORE UPDATE`, so all three checks run in the same row write. Handles every current AND future UPDATE call site without per-site discipline.

**Live DB state:**
- Smart trigger applied to HGPG Core (`ioypqogunwsoucgsnmla`) via Supabase MCP. Migration at `supabase/migrations/20260512_cma_reports_trigger_smart_freshness.sql`.
- Backfill UPDATE ran on the 47 stale rows → **0 stale / 47 fresh / 5 null** (unchanged). Cache genuinely working now.

Update for [[project_cma_dual_supabase]] context: future writers to `cma_reports` don't need to remember to bump `narrative_generated_at` — the trigger handles it based on which columns actually changed.

---

## Previous session: 2026-05-12 — Narrative cache shipped

PR #42 (`polish-narrative-cache`) — merged + deployed (READY, `dpl_FVoBSUsoKis1thj6ngdfcDZyYhQq`).

**The problem:** every visit to `/seller/packet`, `/buyer/packet`, `/appraiser/packet` was firing a fresh Claude call — even on plain reopen from /history. Burned LLM credits and produced inconsistent prose between views of the same report. With agents about to use this for real, that's not workable.

**Verified up-front:**
- `cma_reports` already had `narrative`, `strategy_prices`, `recommended_price` columns. Missing: `narrative_generated_at`.
- No `editVersion` — used `updated_at`, which is already bumped on every UPDATE by the `cma_reports_set_updated_at` trigger.
- 52 rows total in HGPG Core: 47 with saved narrative, 5 null.
- PDF route is already cached (receives in-memory packet from client; no LLM call).
- Save path was already atomic (single UPDATE writes narrative + strategy_prices + recommended_price).

**What landed:**

1. **DB migration applied via Supabase MCP** to HGPG Core (`ioypqogunwsoucgsnmla`). Added `narrative_generated_at timestamptz`. Backfilled `= updated_at` for the 47 narrative-bearing rows so they hit cache on next view (rather than burning 47 LLM calls). The 5 null-narrative rows stay null and regenerate on first view. **Extended the `cma_reports_set_updated_at` trigger** to also stamp `narrative_generated_at = new.updated_at` whenever the narrative column changes — server-side same-transaction, no browser clock skew can desync the comparison. Migration checked in at `supabase/migrations/20260512_cma_reports_narrative_generated_at.sql`.

2. **`lib/cma/narrative-cache.ts`** (new) — `loadCachedNarrative<T>(reportId)` reads the row and returns the narrative when `narrative_generated_at >= updated_at`, null otherwise. Fail-open on Supabase errors.

3. **Three packet pages** — `useEffect` on mount calls `loadCachedNarrative`. Cache hit → render the saved narrative, skip the Claude call. Cache miss → fall through to existing `generate()`. A `cachedFromSave` flag gates `useSaveNarrative` so the cached narrative isn't echoed back to the row on every view. Appraiser page rebuilds `AppraiserComputed` locally via `buildAppraiserMetrics()` on cache hit (deterministic from ws inputs).

4. **Regenerate always forces a fresh Claude call** regardless of staleness — the agent's escape hatch.

5. **No invalidation logic in `/seller/adjust`** — per spec. The natural `updated_at` bump from autosave is the staleness signal: trigger bumps `updated_at` without touching the narrative column, leaving `narrative_generated_at < updated_at`; next packet view regenerates and the save-back writes both timestamps to the same `now()` server-side so subsequent views cache.

CMA tool is now genuinely production-ready for real-world agent use.

---

## Previous session: 2026-05-12 — Address normalization polish shipped

PR #41 (`polish-address-normalize`) — merged + deployed (READY, `dpl_54X51xHk7vhC729jR43L1rqHn4eQ`).

**The problem:** agents type "215 tyndale ct" and every packet surface (title page, headers, narrative, PDF) renders that exact string. Wanted display as "215 Tyndale Court".

**The verified-up-front constraint:** `mls_property.unparsed_address` is null on **0 / 2,575,875** rows. CLAUDE.md was right. So the canonical address has to be reconstructed from `street_number + street_name + street_suffix + unit_number` — all reliably populated, all already stored in proper case with full-word suffixes by MLS Grid.

**What landed:**

1. **DB migration applied via Supabase MCP** to `wdheejgmrqzqxvgjvfee`. Three RPCs (`mls_subject_detect_v2`, `mls_subject_detect_v2_fuzzy`, `mls_subject_detect_v2_zip_fallback`) now additively return `street_number, street_name, street_suffix, unit_number` at the end of the result tuple. Postgres required DROP+CREATE because the return shape changed; existing callers ignore the new keys (PostgREST is name-keyed). No predicate / ordering changes. Migration also checked into repo at `supabase/migrations/20260512_mls_subject_detect_v2_canonical_parts.sql`.
2. **`lib/cma/addressNormalize.ts` (new)** — exposes `buildCanonicalAddress(parts)` for the MLS match path and `normalizeAddress(raw)` for manual entry / legacy reports. Normalizer is idempotent (verified 14/14 cases including "215 Tyndale Ct Apt 4B" → "215 Tyndale Court Apt 4B" and "5601 N Main St" → "5601 North Main Street"; "215 Dr Phil Lane" left alone because "Dr" isn't the last street token).
3. **`lib/mls/subjectDetect.ts`** — `SubjectMlsRow` gets the four parts fields; `MlsSubjectMatch.context.canonicalAddress` is the constructed display address. `SubjectForm.applyMatch` overwrites `subject.address` with it on a match.
4. **`SubjectForm.maybeAutoFillFromMLS`** — on MLS miss, runs the normalizer over the agent's input.
5. **`lib/cma/store.ts`** — `saveWorkingSet` / `loadWorkingSet` apply the normalizer defensively. Single choke point that cleans the **43+ legacy `cma_reports` rows** on the way out via history reopen, no in-place backfill needed.

CLAUDE.md's "`unparsed_address` is null on every row" stays accurate. Add: "Canonical display address is built from `street_number + street_name + street_suffix + unit_number` via `buildCanonicalAddress()` in `lib/cma/addressNormalize.ts`; the three `mls_subject_detect_v2*` RPCs now return those parts."

---

## Previous session: 2026-05-12 — CMA mobile pass (actually) complete

PR #39 (Enhancement 2) was the "simpler option" wrap-to-second-row header + a body `overflow-x-hidden` safety net. Brian re-tested on iPhone 15/16 Pro Max and it still read as broken: nav looked bolted-on, form inputs visually overflowed the white card, and `overflow-x-hidden` was masking real width violations.

**PR #40 (`enh2-followup-hamburger-and-overflow-audit`) — merged + deployed (READY).**

What changed:

1. **Header → hamburger drawer at <md, inline nav at ≥md.** `components/Header.tsx` is now a client component. Right-side drawer, 48px tap targets, backdrop/link/ESC close, body scroll-lock while open. Inline SVG icons — no new dep.
2. **Input width fixed at the source.** SubjectForm and ManualCompRow inputs got `block w-full`; their label wrappers got `min-w-0`. Without `w-full` inputs rendered at the browser's intrinsic ~150px width inside a ~300px card on mobile — looked broken. `min-w-0` on grid items prevents wide intrinsic content from blowing the column track past the card edge (the actual horizontal-overflow source).
3. **Removed `overflow-x-hidden` from `<body>`.** Underlying violations are gone; future regressions surface at audit time instead of being silently masked.

Deploy: `dpl_H5MPdGpug4Qc2uopVFWEzbzAQX9o`, READY at cma.homegrownpropertygroup.com.

Mobile responsive pass is now actually done.

---

## Earlier same-day: 2026-05-12 — Buyers Guide instrumentation + Manus extraction

### Highlights

**Buyers Guide instrumentation (shipped + merged):**
- Pre-session belief: build Pixel + CAPI + NeverBounce from scratch. Reality: Pixel + CAPI code was on `main` since late April but inert because `VITE_META_PIXEL_ID` was unset (Vite tree-shook the dead path).
- NeverBounce genuinely missing; shipped in commit `5c233ee` on `claude/buyers-guide-instrumentation-nGCLI`, PR merged.
- Pixel ID for Buyers Guide: `1449157226505129`. NeverBounce key: reuse Sellers Guide value.
- Two pre-existing TS errors in `Bonuses.tsx` + `Calculator.tsx` fixed to clear `npm run build` gate (Vercel uses `npx vite build` only so they didn't break production but were blocking the local gate).

**Manus migration kickoff (huge):**
- Discovered during recap that the Vercel rebuild of the Buyers Guide is a **significant downgrade** from the original Manus app. Eight high-value server-side features dropped silently: lead scoring, calculator→FUB custom fields + behavioral tags, exit-intent popup + PDF, server-side PDF gen, bonus unlock tracking, /:agent vanity URLs, /admin, /agent-dashboard, advisor mode UX layer.
- Manus support is unreliable; used the Manus AI agent inside their dashboard as an unofficial export channel.
- **Round 1 (probe):** `package.json`, `App.tsx`, `tsconfig.json` confirmed agent access works.
- **Round 2 (map):** full file tree, server entry, root tRPC router, Drizzle schema.
- **Path B (full source export):** complete zip delivered by agent. 220 files, no node_modules bloat, no `.env` files leaked.
- **Snapshot repo created:** `HGPG1/charlotte-buyers-guide-manus-export` (private). First commit `e107b0e` (220 files, 35k+ insertions, 2.81 MiB push). Repo live at https://github.com/HGPG1/charlotte-buyers-guide-manus-export.
- **Historic FUB API key found** in `scripts/get-fub-users.ts` (`fka_0cyqBUzL20pdO11gGKd6J4jcHrUp1P6wsu`). Probed `/v1/identity` — HTTP 401. Already revoked. No action required.
- **Infra clarification:** Manus uses its own "Forge" storage proxy (Bearer-token HTTP API), NOT AWS S3 directly. The `@aws-sdk/*` packages in `package.json` are dead deps. Port target for Vercel rebuild: Supabase Storage (HGPG Core already provisioned).

### What's pending

**Manus migration:**
1. ⏳ **Path C — DB data export** (NEXT). Prompt staged in `projects/buyers-guide.md`. Highest single-value extraction remaining. Actual rows in `contacts`, `activities`, `quizResults`, `agents` only exist on Manus. Fire BEFORE telegraphing departure.
2. Phase 1 ports to Vercel rebuild (1-2 sessions): lead scoring, calculator→FUB custom fields/tags, server-side PDF gen (PDFKit + Supabase Storage), exit-intent popup, bonus unlock tracking.
3. Phase 2: activities event tracking, quiz results DB storage, AI Chat Box review.
4. Phase 3 (deferred until usage confirmed): /:agent vanity URLs, /admin, /agent-dashboard, advisor mode.
5. Manus 301 — must be configured on Manus's side; defer until Path C is complete.

**Buyers Guide instrumentation finishing:**
1. Add 5-6 Vercel env vars in Production scope (`VITE_META_PIXEL_ID`, `META_PIXEL_ID`, `META_CAPI_ACCESS_TOKEN`, `META_CAPI_TEST_EVENT_CODE`, `NEVERBOUNCE_API_KEY`, confirm `FUB_API_KEY`).
2. Trigger fresh production deploy (env-var change alone doesn't rebuild a Vite app).
3. Run post-deploy bundle-grep gates (commands in `projects/buyers-guide.md`).
4. End-to-end smoke in Meta Test Events to confirm browser+server dedup.
5. Remove `META_CAPI_TEST_EVENT_CODE` from Vercel env once dedup confirmed.

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal. Check next time portal is touched.

### Lessons noted (cumulative)

1. **Code-on-`main` does NOT equal shipped-in-production.** Verify against built bundle, not source.
2. **Build-gate divergence:** `vercel.json` `buildCommand` vs `package.json` build script can drift; TS errors land on main without breaking production. Run `npm run build` locally before merging.
3. **Sandbox push works** for at least the Buyers Guide repo. Stale assumption corrected.
4. **Migration-without-source = downgrade.** Original Manus→Vercel migration ported only buyer-facing UI; eight server-side features dropped silently. Always extract source code + DB schema FIRST when migrating off a platform.
5. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with 2-3 innocuous files; verify against production bundle's `data-loc` strings; escalate to full zip. Keep framing as audit/debug — don't telegraph departure.
6. **AWS SDK deps != AWS usage.** Read the code, don't infer infra from `package.json` alone.
7. **Verify before rotation chaos.** Grep extracted source for secrets, then probe them. The FUB key looked dead and was — saved a multi-system rotation.
8. **`apply_migration` via MCP doesn't update repo.** DB ledger and `supabase/migrations/` are separate sources of truth.
9. **CONTEXT.md drifts.** Reconcile vs project files every couple weeks.
10. **Don't trust assertions about "this is done" without verifying against the live system.** Cost of verification is near-zero.

---

## Previous session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup

### What got done

Reconciled CONTEXT.md against project files (which were ahead), closed six items:

1. **Sherlock 403 on Transaction Manager** — was already marked closed in `projects/transaction-manager.md` on 2026-05-09.
2. **Mac Mini GitHub auth / Listing Report Portal** — already 🟢 in project file.
3. **Exposed GitHub PAT rotation** — closed. Fine-grained brain-app PAT superseded the broad old token.
4. **CMA Engine MLS Grid auto-pull** — already marked mature in project file.
5. **NC office routing in ReZEN builder** — verified live in code.
6. **transaction-pdfs bucket flip → private** — fully closed. Bucket flipped via MCP on 2026-05-09; migration file backfilled to repo on 2026-05-11 (`2eb9794`).

Also: rebuilt `projects/brain-app.md` (had been clobbered with handoff-style content), updated CONTEXT.md "active right now" list.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase. Rate limit 2/hr → 30/hr. Affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app.
- Supabase project renames for hygiene.
