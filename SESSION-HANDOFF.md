<!-- Last Updated: 2026-05-14 -->

# Session Handoff

## Last session: 2026-05-14 — Brain App switched from PAT to GitHub App 🟢

### What got built
- **GitHub App "HGPG Brain Commit"** created in HGPG1 org
  - App ID: `3712986`
  - Installation ID: `132322328`
  - Permissions: Contents read/write, Metadata read-only
  - Installed on: ALL HGPG1 repositories
  - Private key: stored as Vercel env var `GITHUB_APP_PRIVATE_KEY` on brain-app project, local copy at `~/.hgpg-secrets/github-app-private-key.pem` on iMac (chmod 600)
- **`lib/octokit-app.ts`** (new) — GitHub App auth helper that mints fresh installation tokens
- **`lib/github.ts`** (refactored) — brain UI reads/writes now use the App instead of a PAT
- **`/api/external/write`** (refactored) — bearer-authed brain write API now uses the App
- **`/api/external/commit`** (NEW) — bearer-authed endpoint that can commit to ANY HGPG1 repo, not just the brain

### Why
- A PAT got exposed in chat history on 2026-05-13 and all PATs were revoked
- This broke `/api/external/write` (was returning 502 "GitHub read failed")
- Rather than rotate to another PAT, switched to GitHub App auth permanently:
  - Installation tokens auto-rotate every hour (no manual rotation ever needed)
  - One bearer token (`BRAIN_WRITE_TOKEN`) is the only thing to rotate if leaked
  - App can be installed on additional repos in seconds via GitHub UI
  - No PAT exists on disk anywhere in the stack

### Verified working end-to-end
- `GET /api/external/write` returns `authMode: "github-app", configured: true`
- `GET /api/external/commit` returns `authMode: "github-app", configured: true, allowedRepos: "all"`
- Live write to `HGPG1/hgpg-context/_diagnostics/github-app-write-test.md` → HTTP 200, commit `d33d9ec` (file later removed in cleanup commit `1d24043`)
- Live cross-repo commit to `HGPG1/brain-app/_diagnostics/cross-repo-commit-test.md` → HTTP 200, commit `098007e` (file later removed in cleanup commit `6cd2536`)

### Env vars added to brain-app on Vercel (Production)
- `GITHUB_APP_ID` = 3712986 (sensitive)
- `GITHUB_APP_INSTALLATION_ID` = 132322328 (sensitive)
- `GITHUB_APP_PRIVATE_KEY` = full PEM contents (sensitive, multi-line)
- `ALLOWED_REPOS` = (not set, meaning "all HGPG1 repos the App has access to")

### Env vars NO LONGER USED (removed 2026-05-14)
- `GITHUB_PAT` — confirmed removed via `vercel env rm`
- `GITHUB_TOKEN` — was never actually set (code had fallback only)
- `BRAIN_GITHUB_PAT` — was never actually set (code had fallback only)
Final state verified: `vercel env ls | grep -E 'PAT|TOKEN'` returns only `BRAIN_WRITE_TOKEN` (the bearer).

### How Claude sessions write to the brain now
```
POST https://brain.homegrownpropertygroup.com/api/external/write
Authorization: Bearer <BRAIN_WRITE_TOKEN>
Content-Type: application/json
{
  "path": "SESSION-HANDOFF.md",
  "content": "<full file contents>",
  "message": "optional commit message"
}
```

### How Claude sessions commit code to other HGPG1 repos now
```
POST https://brain.homegrownpropertygroup.com/api/external/commit
Authorization: Bearer <BRAIN_WRITE_TOKEN>
Content-Type: application/json
{
  "repo": "hgpg-transaction-manager",
  "path": "app/api/foo/route.ts",
  "content": "<full file contents>",
  "message": "optional commit message",
  "branch": "main"
}
```
Same bearer token for both endpoints. Author of commits is Brian McCarron <brian@homegrownpropertygroup.com>.

### Project status updates
- `projects/brain-app.md` — needs update: auth model section + new commit endpoint docs
- `infrastructure.md` — needs update: PAT references should be removed, GitHub App documented

### Pickup notes for next session
- **For Claude:** You can now write directly to any HGPG1 repo via `/api/external/commit`. No more "push this from your Mac" handoffs for routine code changes. Brian still pushes from Mac when he's editing locally and committing his own work — that's normal git, not changed by this work.
- **Rotation playbook:** if `BRAIN_WRITE_TOKEN` ever leaks: `vercel env rm BRAIN_WRITE_TOKEN production && vercel env add BRAIN_WRITE_TOKEN production` then redeploy. Update Brian's memory in chat with the new token. ~60 seconds end to end.
- **If you need to revoke the GitHub App itself** (e.g. the private key leaks): go to github.com/organizations/HGPG1/settings/installations/132322328, click "Suspend" or generate a new private key, then update `GITHUB_APP_PRIVATE_KEY` in Vercel.
- **PAT cleanup:** Brian killed all PATs on 2026-05-13. The brain-app code no longer references them. If any other HGPG1 app still has dead PAT env vars, those can be cleaned up at leisure but won't break anything.
- **Diagnostic files** removed from both repos in cleanup commits `1d24043` (hgpg-context) and `6cd2536` (brain-app). Brain is clean.

## Cleanup sweep (later same day, 2026-05-14)

- Deleted `_diagnostics/` directories from both repos via gh CLI (commits `1d24043` and `6cd2536`)
- Removed stale `GITHUB_PAT` env var from brain-app Vercel project (production + preview scopes)
- Confirmed `GITHUB_TOKEN` and `BRAIN_GITHUB_PAT` were never actually set — the code had fallbacks for them but no env vars existed
- Re-swept the brain for stale references and patched CONTEXT.md, SESSION-HANDOFF.md, and projects/brain-app.md accordingly
- Final state: zero PAT references in active env vars, code, or active documentation. Only historical mentions remain in bug-history sections (intentional record).
