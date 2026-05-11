<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Last session: 2026-05-11 — Brain drift cleanup

### What got done
- Confirmed Sherlock 403 on Transaction Manager is resolved (was already marked closed in `projects/transaction-manager.md` on 2026-05-09, but CONTEXT.md still listed it as a blocker).
- Brian noted the fix was likely security/auth related but did not have specifics. No root-cause logged — if it recurs, start with API key scope.
- `CONTEXT.md` updated: Sherlock moved from blockers to recently completed. Last-updated bumped to 2026-05-11.

### Pickup notes for next session
- Brain index is now in sync with project files. Worth a periodic reconciliation pass like this — drift creeps in when project files get updated but CONTEXT.md doesn't.
- Most natural pickups still on the board:
  - GitHub auth on Mac Mini (still blocks Listing Report Portal)
  - Exposed GitHub PAT rotation (security hygiene)
  - `transaction-pdfs` bucket flip to private — PR not yet opened on branch `claude/transaction-pdfs-private-AqXkA` (see `projects/transaction-manager.md` for full order of operations)
  - CMA Engine MLS Grid auto-pull (active build)
  - NC office routing verification in ReZEN builder (open item in TM)

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
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration" (verify)
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS" (verify)
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation" (verify)
- Supabase `HGPG Core` redirect URLs added:
  - `https://brain.homegrownpropertygroup.com/**`
  - `http://localhost:3000/**`
  - (Existing tools.hgpg entries left intact)

### Deferred / Phase 2 for brain-app
- iPhone smoke test (CodeMirror + iOS soft keyboard scroll behavior)
- Cooper Hewitt self-hosted (currently falling back to system sans)
- File rename and delete
- Diff view before save
- Cross-file search
