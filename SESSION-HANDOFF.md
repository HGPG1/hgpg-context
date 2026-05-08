<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Most recent session: 2026-05-08 (overnight), CMA math bundle PR #31 🟢

### What shipped

PR #31, math bundle for CMA Engine: feature parity (Bug 1) + outlier symmetry (Bug 3) + strategy band cap (Bug 11). https://github.com/HGPG1/hgpg-cma-tool/pull/31

- **Bug 1 feature parity:** Auto-Find comps now arrive with `hgpgFeatures` populated by a data-first then remarks-fallback inference pass. `finished-basement` reads `mls_property.below_grade_finished_area > 0` first, falls back to remarks regex when null. `in-law-suite`, `primary-on-main`, `screened-porch`, `chef-kitchen` stay remarks-only. Adjustments engine emits per-feature lines (parity = $0 visible line, subject-only = +rate, comp-only = -rate, neither = no line) instead of one combined "Subject features" line.
- **Bug 3 outlier symmetry:** counter-cluster protection in `detectWithinBandOutliers`. Groups of 2+ flagged comps within 5% of each other AND on the same side AND >25% from cluster median get unflagged together. The Walnut Creek $610K / $595K Soft Shell pair is the verified case.
- **Bug 11 strategy band cap:** shared `lib/cma/strategy.ts` caps Aspirational to `MIN(PMV * 1.05, High - $5K buffer)` and Event to `MAX(PMV * 0.95, Low + $5K buffer)`. Degenerate-band escape hatch when band width < 4%: Aspirational gets at least PMV * 1.02; Event at most PMV * 0.98. Both consumers (seller packet route + PDF one-pager) call the shared function.
- New `mls_property` columns selected: `above_grade_finished_area`, `below_grade_finished_area`, `below_grade_unfinished_area` (the latter two also serve PR 5 GLA/basement separation when that ships).

### Cumulative CMA ship list this session

#25 Bug 5 bounds invariant, #26 Bug 6 active weighting, #27 Bug 4 anchor sanity, #28 Bug 7 self-listing suffix tolerance, #29 Bugs 8+9 distance-tiered cascade + per-comp distance/direction, #30 cross-state comp deprioritization, #31 math bundle (feature parity / outlier symmetry / strategy band cap). All live on `cma.homegrownpropertygroup.com`.

### Pickup notes (CMA)

- PR 5 (Bug 2, GLA / basement separation) is the next CMA work and gets simpler thanks to the structured Carolina MLS sqft fields already wired up by PR #31.
- Saved test reports (Candlestick `3a84f12c`, Redwine, Medlin, the original `8023175d` Candlestick) still hold pre-fix numbers. Re-saving each via `/seller/adjust` is required to apply the math, comp-set, cross-state, feature-parity, and strategy-cap changes to the historical rows.
- See `projects/cma-engine.md` for the full architecture snapshot post-bundle.

---

## Earlier session: 2026-05-08, NeverBounce wired into Home Grown Selling Score 🟢

### What shipped

NeverBounce real-time email validation is live on the Home Grown Selling Score v2 lead form at `sellersguide.homegrownpropertygroup.com/home-selling-score/`. Mirrors the pattern shipped on `newconstruction.homegrownpropertygroup.com/incentives` 2026-05-05.

Three production deploys this session, all to `charlotte-sellers-guide-vercel`:

1. **`7d86384`** , feat(score): wire NeverBounce email validation. New `api/validate-email.js` ESM proxy, inline status UI on the lead form, FUB forward of `email_validation_status` + `email_validation_flags`, Supabase persistence via `seller_assessments_add_email_validation_2026_05_08` migration (added `email_validation_status` text + `email_validation_flags` jsonb + partial index).

2. **`3133a81`** , fix(score): re-validate on edit + drop Category Breakdown. Two fixes after Brian smoke-tested: (a) input handler now calls `scheduleNbCheck()` so corrections re-validate without re-tabbing, debounced 500ms inside the function. (b) Removed the Category Breakdown bars from results page entirely. The 80-cap curve made all-100% bars contradict the 80 score, score + tier + copy carry the message instead.

3. **`b8b4e25`** (or whatever it gets), fix(score): clear inline display on setNbStatus so class rules win after idle. Sneaky bug, `setNbStatus("idle")` set `el.style.display = "none"` as an inline style. When the API later resolved with valid/warning/error, the className changed but the inline display:none won, hiding the result. Fix is to clear the inline display first, only re-apply for idle.

### Smoke test results (all passed, 2026-05-08)

- Endpoint direct: `curl -X POST /api/validate-email` returns `{"result":"valid","flags":["has_dns","smtp_connectable"]}` for real email
- Form: typo `someone@gmial.com` shows yellow warning (allows submit)
- Form: edit-in-place re-validates after 500ms (after the inline display fix)
- Form: real email shows green check
- Form: bogus domain shows yellow "we couldn't fully verify" (the `unknown` result, fail-open behavior, intentional, leaving as-is)

### Brain updates this session

- `projects/neverbounce-validation.md` flipped to 🟢 live on /incentives + Home Grown Selling Score
- `CONTEXT.md` row promoted, status updated, recently completed bullet added
- `projects/sellers-guide.md` got a NeverBounce section + 3 new commit references
- This SESSION-HANDOFF.md replaces the earlier CMA wrap

### Vercel infra change

- New env var `NEVERBOUNCE_API_KEY` in `charlotte-sellers-guide-vercel` project (Sensitive, Production + Preview, no Dev pull). Same key as the new construction project.

### What's NOT yet done

- Phase 2 NeverBounce rollout to TM, marketing analyzer, Signature, Buyers Guide, ~30 min/site each
- Domain blocklist (Stephen Cooley, KW, Compass, etc.) deferred until 50+ leads gathered
- Meta Custom Conversion registration for `ScoreCompleted` still pending in Meta Events Manager

### Pickup notes for next session

- The NeverBounce pattern is now well-tested across 2 sites. Phase 2 rollout is straightforward, copy `validate-email.js` to each project, wire blur+input handlers, add env var. Use sellers-guide as the reference, it has the bug fixes the new construction site doesn't yet have (sticky-hide fix, re-validate on edit). Worth backporting both fixes to charlotte-new-construction-nextjs at some point.
- The Category Breakdown removal is a UX win, score circle + tier + copy → straight to lead form. Cleaner conversion path, less to misread.
- FUB AI Agent smoke test status remains UNCLEAR. First action next session, check whether Viktor's Automations 2.0 work is done, query `fub_agent_message_drafts` for stale `pending_review` rows from 2026-05-07, ask Brian directly. Until resolved, `agent_enabled` stays false.
- Don feedback on Home Grown Selling Score v2 still pending real-world data, let it run a few days.
- PropStream Caller decision still parked, decommission vs revive vs extract-and-kill.

### Lessons learned this session

- **Brand: no em dashes anywhere.** Got bitten on this in CONTEXT.md and SESSION-HANDOFF.md earlier today, applied a global strip across the brain. Pattern: write with hyphens, commas, parens, period breaks. Em dashes leak in via dashes typed on the Mac (auto-replace) and via Claude output, so worth a final em dash sweep before committing brain files.
- **Inline styles vs classes for visibility toggles.** Setting `style.display = "none"` then later changing the className without clearing the inline style is a classic CSS specificity gotcha. Fix is either always clear the inline style first, or use class-only visibility (e.g., a `.hidden` class). Hit on the NeverBounce status line, fix shipped in commit `b8b4e25`.
- **The `unknown` result from NeverBounce is correctly fail-open.** Bogus domain → `unknown` → yellow warning + allow. The cost of blocking real leads (where NB confidence is below threshold for whatever reason) is much higher than the cost of a bogus submission slipping through, since FUB tags + downstream filtering catch them anyway.
- **Brain auto-write loop now mature.** Multiple sessions can run in parallel without stomping. Today, this NeverBounce session and a CMA session ran simultaneously; both wrote to brain via /api/external/write, no conflicts. SESSION-HANDOFF.md is still last-write-wins and that's fine, the project files are where state should live.
