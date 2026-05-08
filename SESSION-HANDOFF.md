<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Most recent session: 2026-05-08 (evening) — CMA adjustment engine bugs PR 1-3 shipped 🟢

### What got built (PR 1-3 of 6)

Three of the six CMA adjustment-engine bug fixes from `projects/cma-engine-bugs-2026-05-08.md` shipped to production in sequence. All squash-merged, all on production at `cma.homegrownpropertygroup.com`.

- **PR #25 — Bug 5 (confidence bounds invariant):** https://github.com/HGPG1/hgpg-cma-tool/pull/25 — In `lib/cma/pmv.ts`, after the p25/p75 percentile bounds are computed, enforce `low < anchor < high` (all finite). On invariant failure, substitute `anchor*0.93 / anchor*1.08` and force confidence to a new `"low"` band. Adds `"low"` to the `PmvConfidence` union, threaded through `PmvCard` (red badge "Low (manual review)") and `pmv-narrative.ts` (dedicated copy).
- **PR #26 — Bug 6 (active weighting + zero-closed banner):** https://github.com/HGPG1/hgpg-cma-tool/pull/26 — Counts closed comps once at the top of `computePmv`. When `closedCount === 1`, drops active weight from `0.08` to `0.03` so five actives sum to `0.15` against a `1.0` sold instead of `0.40`. When `closedCount === 0`, sets new `pmv.noClosedCompsWarning: true` flag, forces confidence to a new `"very-low"` band, and renders a red banner on `/seller/adjust`. Appraiser narrative gets a "not a comparable-sales analysis" line.
- **PR #27 — Bug 4 (anchor sanity check):** https://github.com/HGPG1/hgpg-cma-tool/pull/27 — After `computePmv` settles the anchor, scans the post-exclusion math set for any comp within `max($50K, anchor * 5%)` of the anchor. If none qualify, sets new `pmv.anchorSyntheticWarning: true`, forces confidence to `"low"` (unless `"very-low"` is already in play), and renders a red banner on `/seller/adjust` ("Weighted PMV does not correspond to any individual comp"). Seller narrative gets a manual-review line.

### Bug evidence captured pre-fix (HGPG Core Supabase, `cma_reports`)

Confirmed the pre-fix state of all three test reports via `execute_sql` before merging:

- `d2a1710a-ae49-4a44-acc0-04fd9f55b376` (504 redwine st): low=$381,760, anchor=$374,028, high=$381,760, confidence "tight" (Bug 5 collapsed-bounds case AND anchor below low — invariant violation)
- `9753a990-8776-421f-ad86-b086fd0472ec` (5601 Medlin Rd): low=$331,500, anchor=$376,388, high=$331,500, confidence "wide" (Bug 5 anchor-above-high case)
- `8023175d-37c8-45d7-95da-b9245abe4761` (6022 Candlestick Ln): low=$1,090,000, anchor=$1,172,235, high=$1,287,000, confidence "wide". Closest math-set comps ~$1,107K and ~$1,237K, both more than $58.6K (5% of anchor) from $1,172K — Bug 4 banner will fire on regenerate.

### Verification status

- Production deploys for PRs #25 and #26 are READY at the time of this writeup. PR #27 was BUILDING when the handoff was written; expected READY within ~90s of merge.
- Live UI re-verification on the saved Redwine, Medlin, and Candlestick reports is **NOT YET DONE** — saved reports cache the math output; per `CLAUDE.md` regenerating only re-runs the AI narrative, so producing a clean Redwine/Medlin/Candlestick result requires opening `/seller/adjust` and re-saving each report. Brian to verify next time he's at a desk.

### Pickup notes

- **PR 4 (Bug 1 — feature parity) is paused awaiting Brian's go.** This is the biggest dollar-impact fix, and per the original prompt Claude was instructed to stop here for confirmation before starting it. Open `projects/cma-engine-bugs-2026-05-08.md` for spec, then run a fresh Claude Code session on `~/Documents/hgpg-cma-tool` with the explicit go-ahead.
- PR 5 (Bug 2 — GLA / basement separation) requires schema additions to `cma_reports.subject_summary` jsonb shape and the Subject input form. Original prompt instructed Claude to also pause before this one because it touches data shape.
- PR 6 (Bug 3 — outlier symmetry) is the smallest fix and lowest priority. Cluster-based outlier evaluation (`±5%` of each other AND `>25%` from anchor → keep both at standard weight or drop both, never split).
- Saved Redwine + Medlin + Candlestick reports still hold the buggy numbers in the database. Regenerating them will not refresh the math — the agent has to walk back into `/seller/adjust` and re-save.
- The new `PmvConfidence` band ladder is now `tight | moderate | wide | low | very-low`. Override priority in `computePmv` is documented inline at the override site.
- `PmvResult` shape gained two boolean flags: `noClosedCompsWarning`, `anchorSyntheticWarning`. Backward-compat for older saved reports without these fields: undefined coerces to false in the React banner conditionals.

---

## Earlier session: 2026-05-08 (afternoon) — Strategic queue refresh + brain hygiene 🟡

### What got done
- Verified Pixel/CAPI rollout status across all consumer sites:
  - ✅ charlotte-new-construction (original implementation)
  - ✅ charlotte-buyers-guide (shipped 2026-04-29, full 6-event chain + dedup)
  - ✅ charlotte-sellers-guide (shipped 2026-05-07, pixel ID 861295553661596, CAPI mirror, FUB UTM custom fields, AssessmentStarted + Lead + ScoreCompleted + QuizStarted + QuizCompleted, Meta-bypass path)
  - ❌ Signature (still pending — small open item)
  - ❌ TM + marketinganalyzer (intentionally skipped — internal/PIN-gated audiences)
- Pulled forward the locked queue from the 2026-05-06 strategic review (was lost from prior handoff)

### Pickup notes for next session

**Locked queue (post Brain App + Pixel rollout)**
1. ⏳ Sellers guide Pixel QA — close out before treating as fully done:
   - QA Path A: organic walkthrough of /home-selling-score confirming PageView + AssessmentStarted + Lead + ScoreCompleted in Test Events with browser+server dedup
   - QA Path B: Meta-bypass walkthrough with ?utm_source=meta confirming phone-required + skip 6-digit verify + FUB lead with meta-bypass tag
   - Register ScoreCompleted as Custom Conversion in Events Manager (Lead category, scoped to URL contains home-selling-score)
   - Wire Meta Lead Ads to FUB native integration for Instant Form path
   - After QA passes: remove META_TEST_EVENT_CODE env var from Vercel
2. 🎯 Buyer Alerts — first piece of MLS Dashboard Suite (#8 v2, ~4-5 hr)
   - LLM criteria parser (natural language to structured filters)
   - Matcher cron against fresh mls_property
   - Alert delivery via existing LoopMessage iMessage pipeline from TM
   - UX bottleneck is criteria capture flow — frictionless input is the make-or-break
3. 📊 Market Intelligence — second piece of MLS Dashboard Suite (~3-4 hr, leans on existing TM Recharts)
4. 🔍 Off-market / Expireds Finder — third piece (~5-6 hr, replaces the external scraper Brian is paying for)
5. 🤖 FUB AI Agent rebuild — biggest project, gated on Sendblue eval first
   - Sign up for Sendblue free sandbox
   - Test FUB integration (dashboard → Integrations → Follow Up Boss)
   - Book sales demo for AI Agent and Blue Ocean plans
   - Get LoopMessage second-sender quote for comparison
   - 30-45 min scoping session writing real spec (channel mix, decision authority, stale-lead scope, compliance)

**Smaller open items**
- Signature Meta Pixel + CAPI (~30-45 min, copy buyers guide pattern)
- MLS Grid Media kickoff (parked alongside .net→.com migration)

**Parked**
- #3 hgpg-transaction-monitor (audit checklist required)
- #6 deal notes UI (after CMA battle testing)
- MLS Grid Media
- #14 .net→.com migration
- Listing Report Portal MLS enrichment (low ROI — portal stable since 2026-04-03)

### Why MLS Dashboard Suite is the next big lever
MLS Grid is live with 2.57M rows replicating via cron, but only the CMA Engine consumes it. The infrastructure is already paid for, so every new tool built on top is cheap marginal value. Buyer Alerts is the highest deal-velocity tool in that suite.

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

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
