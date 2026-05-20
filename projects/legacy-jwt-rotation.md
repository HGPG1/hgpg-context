<!-- Last Updated: 2026-05-20 -->

# Legacy HS256 JWT Rotation Brief (HGPG Core)

## Why this exists

On 2026-05-20 the HGPG Core `service_role` JWT was accidentally pasted in full into a Claude conversation. The conversation is private to Brian's Anthropic account and not publicly accessible, so the realistic threat surface is narrow — but the key is technically valid and must be rotated.

Initial attempt to revoke the legacy HS256 signing key via the Supabase JWT Keys page returned: *"It's not possible to revoke the legacy JWT secret unless you have already disabled JWT-based legacy API keys."* That message is the whole reason this brief exists — the production apps are still authenticated by JWTs signed with HS256, so revoking the secret directly would 401 every production app simultaneously.

The fix is to issue new-style API keys (`sb_secret_*`), update every Vercel project to use them, redeploy, verify everything works, then revoke the legacy secret. That sequence is in Section 4.

## Current state as of 2026-05-20

**Supabase HGPG Core (`ioypqogunwsoucgsnmla`):**
- Legacy HS256 signing secret: still active, signs both anon and service_role legacy JWTs
- Legacy anon JWT (`eyJhbGc...`): not in active use — production has migrated to `sb_publishable_*` keys
- **Legacy service_role JWT: still in active use everywhere, this is the leaked credential**
- New-style keys already issued on the API Keys page:
    - Publishable keys: `default`, `new_p_key`
    - Secret keys: `newkey` (one key already exists, may be in use by something, must audit before assuming it's free to repurpose)

**Apps in scope:**
- `hgpg-transaction-manager` — TM, deployed at closings.homegrownpropertygroup.com. Local `.env.local` confirmed: anon already on `sb_publishable_*`, service_role still legacy.
- `hgpg-cma-tool` — CMA Engine, deployed at cma.homegrownpropertygroup.com. Local `.env.local` confirmed: service_role still legacy. ALSO — local `SUPABASE_URL` points at `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS) which is wrong; production Vercel env has the correct HGPG Core URL. Fix the local drift as part of this session.
- `brain-app` — brain.homegrownpropertygroup.com. Reads service_role from `internal_secrets`-backed flow.
- `charlotte-sellers-guide-vercel` — uses HGPG Signature + Relocation (`fkxgdqfnowskflgbuxhm`), not HGPG Core. **Out of scope** for this rotation but verify before assuming.
- `south-charlotte-report` — may use HGPG Core via service_role for the daily report pipeline. **Audit before assuming.**
- `hgpg-team-dash` — uses HGPG Listing Reports + MLS auth; verify it does not also touch HGPG Core.
- Anywhere else with `SUPABASE_SERVICE_ROLE_KEY` env var pointing at HGPG Core. The audit step in Section 2 covers this.

## Approach

The rotation is small but the verification surface is wide. Order matters:

1. **Audit phase** — find every place the legacy service_role JWT is used. Don't trust the brain; verify against Vercel and local repos.
2. **Provision phase** — create or identify a clean new `sb_secret_*` key with a clear name and description.
3. **Cutover phase** — update Vercel env vars project by project, redeploy, verify with curl before moving to the next project.
4. **Revocation phase** — once every project is verified on the new key, revoke the legacy HS256 secret on the Supabase JWT Keys page.
5. **Cleanup phase** — update local `.env.local` files (do NOT commit), update the brain.

Each phase has a stop-and-verify before moving to the next. This is interactive Brian-and-Claude work in a chat session; not a Claude Code task. Claude Code is the wrong tool because the work is mostly Vercel dashboard navigation + Supabase dashboard clicks + curl verifications, not repo refactoring.

## Section 1 — Pre-flight (5 min)

Before touching anything, confirm the current state still matches this brief.

**1.1 — Confirm legacy key is still valid (from any terminal, not the leaked one):**

    curl -s -o /dev/null -w '%{http_code}\n' \
      -H 'apikey: LEGACY_KEY' \
      -H 'Authorization: Bearer LEGACY_KEY' \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'

Where `LEGACY_KEY` is the leaked key. **Get it from Vercel TM project env vars, NOT from chat history.** Expected: `200` (still valid). If `401`, the legacy was already revoked somehow and the rotation is complete — skip to Section 5 brain update.

**1.2 — List Vercel projects in the HGPG team:**

    vercel projects ls --scope team_FietQPKCmnyioG2n0FdteQCV

Expected: at least `hgpg-transaction-manager`, `hgpg-cma-tool`, `brain-app`, plus others. Note any project name that looks like it could touch HGPG Core.

**1.3 — Confirm the current production keys at the Supabase API Keys page** (https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/api):

- "Publishable and secret API keys" tab: confirm `default`, `new_p_key`, and `newkey` are still listed
- "Legacy anon, service_role API keys" tab: confirm the legacy keys are still there (don't reveal them)

## Section 2 — Audit (15-25 min)

Find every consumer of the legacy service_role JWT for HGPG Core.

**2.1 — Vercel env vars per project.** For each project from Section 1.2:

    vercel env pull /tmp/audit-PROJECTNAME.env --environment=production --scope team_FietQPKCmnyioG2n0FdteQCV --yes
    grep -E 'SUPABASE' /tmp/audit-PROJECTNAME.env | awk -F'=' '{print $1"="substr($2,1,15)"..."}'

Look for any `SUPABASE_SERVICE_ROLE_KEY` starting with `eyJ` (legacy JWT) paired with `SUPABASE_URL` or `NEXT_PUBLIC_SUPABASE_URL` pointing at `ioypqogunwsoucgsnmla.supabase.co`. Those are the projects that need rotation.

**Delete the temp files when done:**

    rm /tmp/audit-*.env

**2.2 — Build the rotation list.** Document each project's:
- Project name and Vercel slug
- Current env var names (some use `SUPABASE_SERVICE_ROLE_KEY`, others may use a different name)
- Production deployment URL (for smoke-test verification)
- Any cron jobs or scheduled tasks that depend on the service role (these may take a deploy + a cron-tick to fully verify)

**2.3 — Confirm the existing `newkey` is or isn't already in use.** This is critical. If `newkey` is already deployed somewhere, we shouldn't recycle it; if it's a leftover from a partial migration that never finished, we can use it. Check by:

- Reveal the `newkey` value in Supabase dashboard
- Grep audit files from 2.1 for the prefix (first 20 chars of `newkey` value)
- If any project's `SUPABASE_SERVICE_ROLE_KEY` starts with that prefix, `newkey` is in active use
- If not, `newkey` is orphaned and can either be revoked or repurposed

## Section 3 — Provision (5 min)

**3.1 — Decide on the `newkey` question from 2.3:**

- If `newkey` is already in active use in some project, leave it alone; create a NEW secret key for this rotation.
- If `newkey` is orphaned, the cleaner move is still to create a new key with a clear name (e.g., `tm-cma-brain-2026-05`) and revoke `newkey`.

**3.2 — Create the new secret key** at the Supabase API Keys page:

- Click "+ New secret key"
- Name it descriptively (e.g., `hgpg-core-service-2026-05`)
- Add a description noting the rotation date and which apps use it
- Copy the value immediately and store securely — it's only shown once

**3.3 — Quick functional test of the new key** from a clean terminal:

    NEW_KEY='paste-here'
    curl -s -o /dev/null -w '%{http_code}\n' \
      -H "apikey: $NEW_KEY" \
      -H "Authorization: Bearer $NEW_KEY" \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'
    unset NEW_KEY

Expected: `200`. If `401`, the new key is broken — go back to Supabase dashboard.

## Section 4 — Cutover (30-45 min)

Update one project at a time, redeploy, verify, move on. Order matters: do the lowest-traffic projects first so any breakage is caught with minimum user impact.

**Suggested order:**

1. `brain-app` (low traffic, single-user, easy to verify)
2. `hgpg-cma-tool` (CMA generation is verifiable end-to-end via UI)
3. `hgpg-transaction-manager` (highest stakes; do last so we know everything else works first)
4. Anything else surfaced in Section 2

**For each project, repeat this sequence:**

**4.x.1 — Update Vercel env var:**

Via Vercel dashboard: Project → Settings → Environment Variables → find `SUPABASE_SERVICE_ROLE_KEY` → Edit → paste new value → Save. Make sure it's applied to all three environments (Production, Preview, Development) or just Production depending on what was there before.

Via CLI alternative (if the dashboard is slow):

    cd ~/Documents/PROJECT_DIR
    vercel env rm SUPABASE_SERVICE_ROLE_KEY production --yes
    echo 'NEW_KEY_VALUE' | vercel env add SUPABASE_SERVICE_ROLE_KEY production

**4.x.2 — Redeploy:**

Vercel only injects env vars at build time. The easiest redeploy:

    cd ~/Documents/PROJECT_DIR
    vercel --prod

Wait for the build to complete (1-3 min depending on project).

**4.x.3 — Verify via app endpoint.** Don't just check that the site loads — exercise the path that uses the service role:

- **brain-app:** load https://brain.homegrownpropertygroup.com, confirm magic link login works (uses service role for auth-related queries)
- **CMA:** generate a CMA at https://cma.homegrownpropertygroup.com (full end-to-end exercises service role on packet persist)
- **TM:** load https://closings.homegrownpropertygroup.com, open any deal, edit any milestone (writes go through service role helper in `lib/createTransaction.ts`)

**4.x.4 — If anything 401s or errors:** roll back this project only by re-setting the env var to the legacy JWT, redeploy, and stop. Tell Brian. Don't proceed to the next project until the failed one is understood.

## Section 5 — Revocation (5 min)

**Only after every project in Section 2 is verified on the new key:**

**5.1 — Final confirmation curl with the legacy key:**

    curl -s -o /dev/null -w '%{http_code}\n' \
      -H 'apikey: LEGACY_KEY' \
      -H 'Authorization: Bearer LEGACY_KEY' \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'

(Get `LEGACY_KEY` from a saved copy or from the Supabase dashboard "Legacy anon, service_role API keys" tab → service_role row → Reveal.)

Expected: still `200`. We're about to make this fail.

**5.2 — Disable legacy API keys first.** This is the precondition Supabase enforced earlier. On the Legacy anon, service_role API keys tab, find the "Disable legacy API keys" button at the bottom → click "Disable JWT-based API keys". Confirm.

**5.3 — Confirm legacy is dead:**

Re-run the curl from 5.1. Expected: `401` (or `403`). If still `200`, the disable didn't apply — refresh the page, check the API Keys page state, retry.

**5.4 — Revoke the legacy HS256 signing key.** Now go to https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/jwt → "Previously used keys" → find the Legacy HS256 (Shared Secret) row → three-dot menu → Revoke. Confirm.

**5.5 — Sanity check production once more:**

- Load TM, open a deal, edit a milestone — should work
- Generate a CMA — should work
- Load brain app — should work

If any of these fail at this point, the legacy key was still being used somewhere the audit missed. Use Vercel env vars to swap that project to the new key urgently, redeploy. The legacy is now dead so there's no "rollback" — only "forward fix."

## Section 6 — Cleanup (10 min)

**6.1 — Update local `.env.local` files** on Brian's Mac. **Do NOT commit these — they're gitignored.**

    cd ~/Documents/hgpg-transaction-manager
    # Edit .env.local: replace the SUPABASE_SERVICE_ROLE_KEY value with the new sb_secret_* value

    cd ~/Documents/hgpg-cma-tool
    # Edit .env.local: replace SUPABASE_SERVICE_ROLE_KEY
    # ALSO fix SUPABASE_URL — current value points at wdheejgmrqzqxvgjvfee (Listing Reports), should be ioypqogunwsoucgsnmla (HGPG Core).

**6.2 — Close any open Terminal windows** that had `SUPABASE_SERVICE_ROLE_KEY` exported from the previous session. The export only lives in that shell's memory, but closing the window guarantees it's gone.

**6.3 — Update the brain.** Append a new entry to the top of `SESSION-HANDOFF.md` via `POST /api/external/write` covering:

- Rotation completed, legacy HS256 secret revoked
- New `sb_secret_*` key in use across N projects
- Audit list of projects rotated
- Any surprises found during the audit (orphaned keys, misconfigured `.env.local` files, etc.)
- Cluster C session can now resume (it was paused on this)

Also archive this brief or leave it in place with a final "STATUS: COMPLETE" line at the top.

## Estimated total

- Section 1 pre-flight: 5 min
- Section 2 audit: 15-25 min
- Section 3 provision: 5 min
- Section 4 cutover: 30-45 min (depends on how many projects)
- Section 5 revocation: 5 min
- Section 6 cleanup: 10 min

**Total: 70-95 min of focused work.** Less if no surprises in the audit; more if a missed project surfaces during revocation.

## Stop conditions

Stop and ask Brian if:

- Section 2 audit surfaces a project that uses the legacy key in a way that's hard to redeploy (e.g., a Lambda, a non-Vercel host, a cron in a different system)
- The existing `newkey` secret in the Supabase dashboard turns out to be in production use by a project we didn't know about
- Any verification curl in Section 4 returns `200` for a project we just updated (means the new key was rejected — possible mismatch between client expectation and key format)
- Section 5.2 "Disable legacy API keys" returns an error or won't let us click confirm
- After Section 5.4 revocation, any production smoke test fails

## Rollback plan

**Before Section 5.4** (revocation): trivial — re-set Vercel env var to legacy JWT on any failing project, redeploy, you're back. The legacy key is still valid until revoked.

**After Section 5.4 revocation**: the legacy key is dead, there's no rollback. The only path is forward: find the missed project, update its env var to the new sb_secret_, redeploy. The revocation is fast but the recovery from a missed audit is slow — that's why Section 2 (audit) is critical and Section 4 (cutover) goes one project at a time with verification at each step.

## Things NOT in scope

- Don't tackle Cluster C in the same session. Rotate first, then Cluster C in a fresh session.
- Don't migrate the anon side of any project that's still on legacy. The current state has TM and CMA already on `sb_publishable_*` for anon; if any other project surfaces with legacy anon, document it but don't touch it during rotation.
- Don't try to rotate the JWT signing key for HGPG Listing Reports + MLS, HGPG Signature + Relocation, HGPG FUB Integration, or Suna projects. The leak only affects HGPG Core.

## Key references

- Brain: https://brain.homegrownpropertygroup.com (web editor) or `~/Documents/hgpg-context` (local clone)
- This brief: `projects/legacy-jwt-rotation.md` in `HGPG1/hgpg-context`
- Supabase project: `ioypqogunwsoucgsnmla` (HGPG Core)
- Supabase API Keys page: https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/api
- Supabase JWT Keys page: https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/jwt
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- Brain APIs: `https://brain.homegrownpropertygroup.com/api/external/{read,write,commit,log}` with bearer token in keychain
- Cluster C Path 3 brief (next session after rotation): `projects/cluster-c-path-3-brief.md`
