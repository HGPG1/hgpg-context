<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — CMA Engine: Enhancement 1 (autosave) + Math bundle Bugs 6/12/13 shipped

### What shipped today

**Enhancement 1 — autosave on /seller/adjust + flip to 'presented' on PDF export.** PR #34, commit `701ed86`. Every edit (rate, feature, line toggle, condition tier, agent-include, swap) bumps an `editVersion` counter; a 500ms-debounced effect writes via the existing `saveCmaReport` path. 3-attempt exponential backoff (250/500/1000ms). "Draft — Saving / Saved HH:MM / Save failed" banner under the page eyebrow. Seller / buyer / appraiser packet pages each flip `cma_reports.status` from draft → presented after successful PDF download. No schema migration (status column already had DEFAULT 'draft').

**Math bundle — Bugs 6 / 12 / 13.** PR #35, commit `00e41d4`. All three fixes in `lib/cma/{outlierFlag,pmv,adjustments,flags,types,adjustment-defaults}.ts`.

- **Bug 6 — outlier symmetry.** New third detector pass after `removeCounterClusters`: scan each remaining flag for an unflagged near-twin (adjustedPrice within 5%, same side of weighted anchor, similarity within 0.10). If cluster median >25% from anchor → both pair-penalized at 0.5x weight (kept in math, NOT auto-excluded). If ≤25% → unflag original (both treated as legit signal). Counter-cluster wins.
- **Bug 12 — singleton-outlier weight cap.** Closed comp triggers when ALL THREE: adjustedPrice >50% above OR >40% below other-Sold median; similarity < 0.65; at least one quality flag (long-market DOM>90, PQS<50, sqft-mismatch>25%, pool-mismatch, cross-state). Caps weight at 0.25 (NOT full exclusion).
- **Bug 13 — time-of-sale appreciation.** New `appreciationRatePerYear` rate (default 5.0%, surfaced in the rates panel). Closed comps get +halfRate% (91-180d) or +fullRate% (181-365d) on raw price; >365d auto-excluded via new `stale-sale` flag.

### Live state

- cma.homegrownpropertygroup.com running both changes. Vercel deploys: Enhancement 1 `dpl_46pzX7iRwpYmrVETWzWrVcignX6t` (29s build), Math bundle `dpl_3unG6moCGWASarFpzHct2diXq32R` (36s build). Both READY.
- Engine bug queue from the original 13-bug sweep is now fully drained. Remaining CMA work is enhancements (autosave already shipped) and quality-of-life items.
- 99.99% of `mls_property` closed rows have `close_date` (1,647,147 of 1,647,282). Bug 13's data dependency is met cleanly — no fallback to `listing_contract_date` needed.

### Smoke test queue for Brian

- Cressingham 5022 / Candlestick 6022: bump a slider, confirm autosave banner cycles Saving → Saved within 1-2s, reload, slider sticks.
- Candlestick re-run: 2915 Cutter Court (Woodhall) should now fire singleton-outlier, weight 0.75 → 0.25, PMV compresses from $1.02M toward $970-985K.
- Bug 6 verification: Soft Shell pair from May 7 run is no longer asymmetric.
- Stale-sale: any closed comp >365 days old shows "Excluded from math · stale sale".

### Critical project knowledge added to /memory

- **`cma_reports` lives in HGPG Core (`ioypqogunwsoucgsnmla`), NOT MLS (`wdheejgmrqzqxvgjvfee`).** The CMA app uses both projects: MLS for comp search, Core for reports + adjustment defaults. Browser autosave uses `NEXT_PUBLIC_SUPABASE_URL` pointed at Core. Easy to query the wrong DB and conclude tables are missing.
- **`'presented'` is canonical for the post-packet status,** not `'published'`. The existing `CmaReportStatus = 'draft' | 'presented' | 'archived'` enum stands. Any future spec language using 'published' generically should resolve to 'presented' in this codebase.

---

## Earlier session: 2026-05-11 — Brain reconciliation + NC office verification

### What got done

Reconciled CONTEXT.md against project files (which were ahead), closed five drifted/stale items:

1. **Sherlock 403 on Transaction Manager** — was already marked closed in `projects/transaction-manager.md` on 2026-05-09. Fix was likely security/auth related; no specifics logged. If recurs, start with API key scope.
2. **Mac Mini GitHub auth / Listing Report Portal** — already 🟢 in project file (2026-05-06 favicon push verified).
3. **Exposed GitHub PAT rotation** — closed. Fine-grained brain-app PAT (scoped to `hgpg-context`, contents:write only) superseded the broad old token. Old token no longer in active use.
4. **CMA Engine MLS Grid auto-pull** — already marked mature in `projects/cma-engine.md` on 2026-05-09. `searchComps()` is load-bearing primary data source.
5. **NC office routing in ReZEN builder** — **verified live**. Code at `app/api/rezen/create-transaction/route.ts:160` uses `txn.state?.toUpperCase() === "NC" ? REZEN_OFFICE_ID_NC : REZEN_OFFICE_ID`, applied via `setOwnerInfo`. Both env vars confirmed set on Vercel Production. NC office ID `924dac0e-91f2-471d-80c4-f06d80fb6d94` pulled from env (not hardcoded — correct pattern).

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal (SocialPostsManager dark navy headings, MagicLinkCard regenerate placement, admin reorder, zero-value stat hiding). Check next time portal is touched.

### Pickup notes for next session

Real build items remaining:

- **`transaction-pdfs` bucket flip to private** — code shipped to branch `claude/transaction-pdfs-private-AqXkA` on 2026-05-09. PR not yet opened, migration not yet applied. See `projects/transaction-manager.md` for full order of operations and rollback. **HELD intentionally as of 2026-05-11** — Brian decision to land it later, not blocked.
- **CMA Engine** awaiting Taylor stress test before declaring production-ready.
- **$395 fee toggle** structural build still parked (commit b9fa0deb).

Non-build:
- .net Google Workspace migration to .com (do not proactively remind).

### Brain hygiene note

When project files get updated, CONTEXT.md drifts. Worth running a reconciliation pass like today's every couple weeks. Quick check: `diff` what CONTEXT.md flags as blockers/active vs. what the actual project files say.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

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
  - Rate limit 2/hr → 30/hr
  - Affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene
- Supabase `HGPG Core` redirect URLs added for `brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search
