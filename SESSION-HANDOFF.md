<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Brain reconciliation + NC office verification

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
