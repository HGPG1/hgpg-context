<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Brain drift cleanup (round 2)

### What got done
- **Sherlock 403** on Transaction Manager confirmed resolved (was already marked closed in `projects/transaction-manager.md` on 2026-05-09, but CONTEXT.md still listed it as a blocker). Fix was likely security/auth related — no specifics logged; if it recurs, start with API key scope.
- **Mac Mini GitHub auth / Listing Report Portal** confirmed resolved (project file already had status 🟢 Live, GitHub auth blocker resolved as of 2026-05-06 favicon push). CONTEXT.md still flagged it.
- `CONTEXT.md` updated: both items moved from blockers to recently completed. Last-updated bumped to 2026-05-11.

### Open thread
- `~/Downloads/fix-admin-page.sh` was flagged on the Listing Report Portal project file as possibly having unpushed UI changes (SocialPostsManager dark navy headings, MagicLinkCard regenerate placement, admin section reorder, zero-value stat hiding). Not verified this session. Check next time the portal is touched.

### Pickup notes for next session
- Brain index now in sync with project files. Drift creeps in when project files update but CONTEXT.md does not — periodic reconciliation passes are worth doing.
- Remaining items on the board:
  - **Exposed GitHub PAT rotation** (security hygiene)
  - **`transaction-pdfs` bucket flip to private** — PR not yet opened on branch `claude/transaction-pdfs-private-AqXkA` (see `projects/transaction-manager.md` for full order of operations)
  - **CMA Engine MLS Grid auto-pull** (active build)
  - **NC office routing verification** in ReZEN builder (open item in TM)
  - **.net Google Workspace migration to .com** (do not proactively remind)

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
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr (Supabase default) to 30/hr (Resend default), can be raised
  - This affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene
- Supabase `HGPG Core` redirect URLs added for `brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search
