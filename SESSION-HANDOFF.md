<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Next session (morning 2026-05-13): Quiz smoke test → Buyers Guide migration Session 3

### Step 1 — Quiz smoke test (5 min, do FIRST)

**Why:** Session 2 was marked shipped after pause at commit `879d263` for a Quiz lifestyle-key mismatch bug. Direct Supabase check 2026-05-12 evening shows `bg_quiz_results` has **0 rows** despite Session 2's smoke test putting 3 rows each in `bg_contacts` + `bg_activities` at the same timestamp. Strong signal the bug was never fully resolved before Session 2 was declared shipped. Verify before Session 3 modifies Quiz to add advisor mode.

**How:**
1. Open https://buyersguide.homegrownpropertygroup.com (incognito ideal so existing UnlockModal state doesn't interfere)
2. Open DevTools → Network tab
3. Navigate to /quiz, complete the 5 questions with test data:
   - Email: `quiz-test-2026-05-13@homegrownpropertygroup.com`
   - Pick any option for each of the 5 questions
4. Submit. Watch Network tab for the `/api/...` call that fires.
5. Check the response — 200 with success, or 4xx/5xx with an error?

**Verification queries (run from Claude session with Supabase MCP after the submission):**

    SELECT id, contact_email, contact_name, buyer_type, lifestyle_priority,
           commute_tolerance, budget_range, property_type, synced_to_fub, created_at
    FROM bg_quiz_results
    WHERE contact_email = 'quiz-test-2026-05-13@homegrownpropertygroup.com'
    ORDER BY created_at DESC;

**Two outcomes:**

- ✅ **Row lands with all 5 fields populated** → quiz bug was fixed, proceed to Step 2 (Session 3)
- ❌ **Row missing OR has null/wrong values in `lifestyle_priority`** → bug is real. Fix Quiz first BEFORE Session 3. Bug is likely a key-name mismatch between client form state and either the API request body or the Supabase column. The fix is small but must be verified end-to-end before stacking advisor mode on top.

If bug is real, the fix is a focused mini-session (~20-30 min): read `src/pages/Quiz.tsx` for what key it submits, read `api/quiz/submit.ts` (or wherever the route lives) for what it expects, reconcile. Run the smoke test again to confirm before moving on.

### Step 2 — Buyers Guide migration Session 3 (~2 hrs)

Only proceed once Step 1 is green.

**Prompt to use:** verbatim in `projects/buyers-guide-manus-migration.md` under the heading "Session 3 — Agent surfaces + Advisor Mode." Copy-paste ready.

**Quick scope reminder for Session 3:**
- Agent routes: `/:agent`, `/advisor`, `/advisor/quiz`, `/advisor/calculator`, `/advisor/neighborhoods`, `/advisor/strategy`, `/advisor/checklist`, `/admin`, `/agent-dashboard`
- Agent detection from URL slug → `bg_agents` lookup → routes new contacts to right FUB user
- Advisor mode context + banner + inline talking points on Quiz/Neighborhoods/Strategy/Checklist
- Agent API routes: `/api/agent/by-slug`, `/api/agent/default`, `/api/agent/performance-stats`
- Protection on `/admin` and `/agent-dashboard` via `?key=X` against `ADMIN_ACCESS_KEY` env
- Talking points content extracted to `src/data/advisor-content.ts`

**Pre-session question Session 3 prompt will ask:** "Do HGPG agents have any external pages/email signatures pointing to `/brian`-style URLs today?" → answer is **no**, the Manus `/:agent` URLs were never shared (zero attributed leads in Manus DB confirms this). Tell the agent to use HGPG-standard slugs (`brian`, `ashley`, `taylor`, `brenda`, `owner` from `bg_agents`) and not worry about backward compatibility with anything in the wild.

---

## End-of-day 2026-05-12 state

### Shipped today (chronological)

1. **Buyers Guide instrumentation** (earlier today) - Pixel + CAPI made live (was inert since April), NeverBounce shipped from scratch. Commit `5c233ee`. PR merged.
2. **Manus AI agent extraction** - 220-file source snapshot in `HGPG1/charlotte-buyers-guide-manus-export`. Live Manus DB confirmed effectively empty (1 fake test lead, 0 quizzes, 0 exit intents, 5 agents) via direct tRPC probe.
3. **Buyers Guide migration Session 1** - Supabase backend (`bg_contacts`, `bg_activities`, `bg_quiz_results`, `bg_agents`), server helpers in `api/_lib/`, FRED + FUB + Storage health endpoints green.
4. **Buyers Guide migration Session 2** - Calculator + Quiz + Exit Intent + Bonus Unlock shipped. **Quiz outcome unverified — see Step 1 above.**
5. **New Construction phone capture** - tiered required/optional across 4 forms, E.164 normalization, per-form helper copy. PR #1 merged (`8bca9ac`).
6. **End-of-day CONTEXT.md reconciliation** - drifted again; full active/completed/blocker rebuild, root cause logged with proposed Phase 2 fix (auto-generate index from project-file frontmatter).

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed Listing Report Portal UI changes. Check next time portal is touched.

### Real build items remaining

- **Buyers Guide migration Sessions 3 + 4** (Session 3 next; Session 4 optional polish)
- **SMS speed-to-lead automation** for New Construction (parked; recommend gate on SMS consent for TCPA defensibility)
- **CMA Engine** - awaiting Taylor stress test
- **$395 fee toggle** (TM) - structural build parked, commit `b9fa0deb`
- **FUB AI Agent** - flip strategy decision pending, 4,340 unscored lead sweep deferred
- **Sellers Guide** - FUB Automation 2.0 on `sellers-guide-2026` tag (blocks ad-spend scaling)
- **Take Manus app down** (no 301) once migration Sessions 3+4 complete
- **`META_CAPI_TEST_EVENT_CODE` cleanup** on Buyers Guide once dedup confirmed in Meta Test Events

### Non-build
- .net → .com Google Workspace migration (do not proactively remind)

### Lessons noted today (key ones)

1. **Code-on-`main` does NOT equal shipped-in-production.** Pixel + CAPI sat on main ~2 weeks with zero `fbq` references in live bundle because `VITE_*` was unset.
2. **Migration-without-source = downgrade — but only when the source app was actually being used.** Always check usage signal before treating missing features as regressions.
3. **AI agents inside SaaS platforms are an unofficial export channel.** Probe with innocuous files; verify against production bundle; escalate to full zip.
4. **When a SaaS AI agent loops, check whether the workspace is archived/disconnected.** GitHub-archived workspace sandboxes the Manus AI agent to dead files.
5. **When AI agent extraction stalls, hit the deployed app directly.** Live tRPC endpoints faster than coaxing an agent.
6. **"Shipped" needs end-to-end verification, not just `npm run build` green.** Today's Quiz situation: Session 2 declared shipped after build passed and partial smoke. Direct DB check at EOD reveals `bg_quiz_results` is empty. Future sessions: "shipped" requires one row of real data in every new table the session touched.
7. **CONTEXT.md drifts structurally.** Two-layer brain, one-layer attention. Phase 2 fix logged: auto-generate index sections from project-file frontmatter.

---

## Previous session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup

Reconciled CONTEXT.md against project files (which were ahead), closed six items: Sherlock 403, Mac Mini GitHub auth, exposed GitHub PAT rotation, CMA Engine MLS Grid auto-pull, NC office routing in ReZEN (verified live in code), transaction-pdfs bucket flip → private (bucket flipped via MCP on 2026-05-09; migration file backfilled to repo on 2026-05-11 as `2eb9794`).

Also: rebuilt `projects/brain-app.md` (had been clobbered), updated CONTEXT.md "active right now" list.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- Resend custom SMTP wired into HGPG Core Supabase (2/hr → 30/hr)
