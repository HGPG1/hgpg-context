<!-- Last Updated: 2026-05-20 -->

# HGPG Core API Key Migration Brief (leaked service_role key)

## STATUS: NOT STARTED — planned for 2026-05-21

## Why this exists

On 2026-05-20 the HGPG Core `service_role` JWT was accidentally pasted in full into a Claude conversation. The conversation is private to Brian's Anthropic account and not publicly accessible, so the realistic threat surface is narrow — but the key is technically valid and must be killed.

The original plan for this was a legacy HS256 JWT-secret rotation. That plan is abandoned. Reason: rotating the legacy JWT secret is a heavy, all-or-nothing operation (it invalidates the anon key and every user session at once), and Supabase no longer supports rotating the legacy secret cleanly anyway. The legacy `anon`/`service_role` keys are permanently coupled to the project's JWT secret.

The better fix — and the one that makes any FUTURE leak a two-click problem — is to migrate HGPG Core onto Supabase's new API key system (`sb_secret_*` / `sb_publishable_*`). Once on the new system, a leaked secret key is rotated by: create new key, swap env vars, delete old key. No JWT-secret rotation, no anon-key collateral, no dropped user sessions. That is the "flip one thing and you're safe" outcome Brian asked for.

This brief migrates HGPG Core only — the project with the leaked key. The other four Supabase projects roll over in a separate follow-up (see "Follow-up" at the end).

## The key insight (why this is a small lift)

- New `sb_secret_*` keys substitute for `service_role` almost everywhere with no code change — `supabase-js` initializes with the new value the same way. It's an env-var swap, not a refactor.
- Rotating a new-style secret key does NOT invalidate user sessions and does NOT touch the anon/publishable side. The coupling that made the old rotation painful is gone.
- HGPG Core's anon side is ALREADY migrated — TM and CMA use `sb_publishable_*` for anon. Only the service_role → secret-key swap remains.

## The one real gotcha — Bearer header audit

New-style keys are NOT JWTs. Supabase rejects a publishable/secret key sent in an `Authorization: Bearer ...` header UNLESS that header value exactly equals the `apikey` header value. `supabase-js` handles this correctly on its own. The risk is any HGPG code that hand-builds an `Authorization: Bearer <service_role>` header manually instead of letting the client library do it. Section 2.4 audits for this. It is expected to be a small or empty set (TM and brain-app use the client library), but it must be checked before cutover, not after.

## Current state as of 2026-05-20

**Supabase HGPG Core (`ioypqogunwsoucgsnmla`):**
- Legacy HS256 signing secret: still active, signs both anon and service_role legacy JWTs
- Legacy anon JWT: not in active use — production has migrated to `sb_publishable_*`
- **Legacy service_role JWT: still in active use everywhere — this is the leaked credential**
- New-style keys already issued on the API Keys page:
    - Publishable keys: `default`, `new_p_key`
    - Secret keys: `newkey` (one secret key already exists — must audit in 2.3 before assuming it is free to use or safe to delete)

**Apps in scope (use HGPG Core service_role):**
- `hgpg-transaction-manager` — TM, closings.homegrownpropertygroup.com. Local `.env.local`: anon already on `sb_publishable_*`, service_role still legacy.
- `hgpg-cma-tool` — CMA Engine, cma.homegrownpropertygroup.com. Local `.env.local`: service_role still legacy. ALSO local `SUPABASE_URL` wrongly points at `wdheejgmrqzqxvgjvfee` (Listing Reports) — production Vercel env is correct; fix the local drift in Section 6.
- `brain-app` — brain.homegrownpropertygroup.com. Reads service_role from an `internal_secrets`-backed flow.

**Audit before assuming in/out of scope:**
- `south-charlotte-report` — may use HGPG Core service_role for the daily pipeline. Audit in Section 2.
- `hgpg-team-dash` — uses HGPG Listing Reports + MLS auth; confirm it does not also touch HGPG Core.
- `charlotte-sellers-guide-vercel` — uses HGPG Signature + Relocation, not HGPG Core. Expected out of scope; confirm in Section 2.

## Approach

Migration is small; verification surface is wide. Order matters:

1. **Audit** — find every consumer of the legacy service_role JWT for HGPG Core, including any manual Bearer-header usage.
2. **Provision** — create or identify a clean `sb_secret_*` key with a clear name.
3. **Cutover** — update Vercel env vars project by project, redeploy, verify with curl before moving on.
4. **Disable + revoke** — once every project is verified on the new key, disable legacy JWT-based API keys (this kills the leaked key), then revoke the legacy HS256 secret.
5. **Cleanup** — update local `.env.local` files (do NOT commit), update the brain.

Each phase has a stop-and-verify before the next. This is interactive Brian-and-Claude chat work — Vercel dashboard navigation, Supabase dashboard clicks, curl verification. Not a Claude Code task.

## Section 1 — Pre-flight (5 min)

**1.1 — Confirm the leaked legacy key is still valid** (from any terminal, not the leaked session):

    curl -s -o /dev/null -w '%{http_code}\n' \
      -H 'apikey: LEGACY_KEY' \
      -H 'Authorization: Bearer LEGACY_KEY' \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'

`LEGACY_KEY` = the leaked key. Get it from the Vercel TM project env vars, NOT from chat history. Expected `200`. If `401`, the legacy keys were already disabled — skip to Section 6 brain update.

**1.2 — List Vercel projects in the HGPG team:**

    vercel projects ls --scope team_FietQPKCmnyioG2n0FdteQCV

Note any project name that could plausibly touch HGPG Core.

**1.3 — Confirm key state at the Supabase API Keys page** (https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/api):
- "Publishable and secret API keys" tab: confirm `default`, `new_p_key`, `newkey` are listed
- "Legacy anon, service_role API keys" tab: confirm the legacy keys are still there (don't reveal them)

## Section 2 — Audit (15-30 min)

**2.1 — Vercel env vars per project.** For each project from 1.2:

    vercel env pull /tmp/audit-PROJECTNAME.env --environment=production --scope team_FietQPKCmnyioG2n0FdteQCV --yes
    grep -E 'SUPABASE' /tmp/audit-PROJECTNAME.env | awk -F'=' '{print $1"="substr($2,1,15)"..."}'

Look for any `SUPABASE_SERVICE_ROLE_KEY` starting with `eyJ` (legacy JWT) paired with a Supabase URL pointing at `ioypqogunwsoucgsnmla.supabase.co`. Those projects need migration.

**2.2 — Build the migration list.** For each in-scope project document: project name + Vercel slug, the exact env var name holding the service role (most use `SUPABASE_SERVICE_ROLE_KEY`, some may differ), production URL for smoke testing, and any cron/scheduled tasks that depend on the service role (these need a deploy AND a cron tick to fully verify).

**2.3 — Resolve the existing `newkey` secret.** Reveal `newkey` in the Supabase dashboard, take its first ~20 chars, and grep the 2.1 audit files for that prefix. If a project's service-role env var starts with that prefix, `newkey` is in active use — leave it alone. If nothing matches, `newkey` is orphaned (a leftover from a partial migration) and can be deleted or repurposed.

**2.4 — Bearer-header audit (the gotcha check).** For each in-scope repo, search for any place that builds an Authorization header by hand rather than letting `supabase-js` do it:

    cd ~/Documents/PROJECT_DIR
    grep -rnE "Authorization.{0,4}Bearer" src app lib api 2>/dev/null | grep -iE 'service|supabase|SERVICE_ROLE'

Expected: few or no hits — TM and brain-app use the client library. Any hit that sends the service_role/secret key as a raw `Authorization: Bearer` value (and is NOT also setting an identical `apikey` header) MUST be fixed before cutover, because the new key will be rejected there. The fix is either (a) route the call through the `supabase-js` client, or (b) add an `apikey` header with the same value. Note each hit and its fix in the migration list. If a fix needs a code commit, do that commit first (via `/api/external/commit`) and let it deploy before Section 4 for that project.

**Clean up audit temp files:**

    rm /tmp/audit-*.env

## Section 3 — Provision (5 min)

**3.1 — Decide on `newkey`** from 2.3: if in use, leave it and create a fresh key; if orphaned, still create a fresh well-named key for this migration and delete `newkey` during cleanup.

**3.2 — Create the new secret key** at the Supabase API Keys page:
- "+ New secret key"
- Name it descriptively, e.g. `hgpg-core-service-2026-05`
- Description: migration date + which apps use it
- Copy the value immediately — shown only once

**3.3 — Functional test the new key** from a clean terminal:

    NEW_KEY='paste-here'
    curl -s -o /dev/null -w '%{http_code}\n' \
      -H "apikey: $NEW_KEY" \
      -H "Authorization: Bearer $NEW_KEY" \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'
    unset NEW_KEY

Expected `200`. Note: this curl sends the key as BOTH `apikey` and `Authorization: Bearer` with identical values, which is the one Bearer usage Supabase allows for non-JWT keys. If `401`, the new key is broken — back to the dashboard.

## Section 4 — Cutover (30-45 min)

One project at a time: update env var, redeploy, verify, move on. Lowest-traffic first so breakage has minimum impact.

**Suggested order:** 1) `brain-app` (low traffic, single-user), 2) `hgpg-cma-tool` (verifiable end-to-end via UI), 3) `hgpg-transaction-manager` (highest stakes — do last), 4) anything else from Section 2.

**For each project:**

**4.x.1 — Update the Vercel env var.** Dashboard: Project → Settings → Environment Variables → `SUPABASE_SERVICE_ROLE_KEY` → Edit → paste new value → Save. Apply to the same environments the old value covered.

CLI alternative:

    cd ~/Documents/PROJECT_DIR
    vercel env rm SUPABASE_SERVICE_ROLE_KEY production --yes
    echo 'NEW_KEY_VALUE' | vercel env add SUPABASE_SERVICE_ROLE_KEY production

**4.x.2 — Redeploy** (Vercel injects env vars at build time):

    cd ~/Documents/PROJECT_DIR
    vercel --prod

**4.x.3 — Verify the service-role path, not just that the site loads:**
- `brain-app`: load brain.homegrownpropertygroup.com, confirm magic-link login works
- CMA: generate a full CMA at cma.homegrownpropertygroup.com (exercises service role on packet persist)
- TM: load closings.homegrownpropertygroup.com, open a deal, edit a milestone (writes go through the service-role helper in `lib/createTransaction.ts`)

**4.x.4 — On any 401/error:** roll this project back only — re-set the env var to the legacy JWT, redeploy, stop, tell Brian. Do not proceed until understood. (Legacy key is still valid at this stage, so rollback is safe.)

## Section 5 — Disable + revoke (5-10 min)

**Only after every in-scope project is verified on the new key.**

**5.1 — Final confirmation the legacy key still works:**

    curl -s -o /dev/null -w '%{http_code}\n' \
      -H 'apikey: LEGACY_KEY' \
      -H 'Authorization: Bearer LEGACY_KEY' \
      'https://ioypqogunwsoucgsnmla.supabase.co/rest/v1/transactions?select=id&limit=1'

Expected `200`. We are about to make this fail.

**5.2 — Disable legacy JWT-based API keys.** On the "Legacy anon, service_role API keys" tab, use the "Disable JWT-based API keys" control at the bottom. Confirm. This is the step that actually kills the leaked key.

**5.3 — Confirm the legacy key is dead.** Re-run 5.1. Expected `401`/`403`. If still `200`, refresh and retry — the disable did not apply.

**5.4 — Revoke the legacy HS256 signing key.** https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/jwt → "Previously used keys" → Legacy HS256 (Shared Secret) row → three-dot menu → Revoke. Confirm. (This is housekeeping — 5.2 already neutralized the leak. Revoking the signing key removes the ability to mint new legacy JWTs at all.)

**5.5 — Production smoke test once more:** TM milestone edit, CMA generation, brain-app load. If anything fails here, the audit missed a consumer — there is no rollback now, only forward fix: swap that project's env var to the new key and redeploy urgently.

## Section 6 — Cleanup (10 min)

**6.1 — Update local `.env.local` files** on Brian's Mac (gitignored — do NOT commit):

    cd ~/Documents/hgpg-transaction-manager
    (edit .env.local: replace SUPABASE_SERVICE_ROLE_KEY with the new sb_secret_ value)

    cd ~/Documents/hgpg-cma-tool
    (edit .env.local: replace SUPABASE_SERVICE_ROLE_KEY; ALSO fix SUPABASE_URL — it points at wdheejgmrqzqxvgjvfee, should be ioypqogunwsoucgsnmla)

**6.2 — Delete the orphaned `newkey`** if Section 2.3 found it unused.

**6.3 — Close any Terminal windows** that had `SUPABASE_SERVICE_ROLE_KEY` or `LEGACY_KEY` exported during the session.

**6.4 — Update the brain.** New entry at the top of `SESSION-HANDOFF.md` via `POST /api/external/write`: migration completed, legacy JWT keys disabled + HS256 secret revoked, new `sb_secret_*` key live across N projects, the audited project list, any surprises (orphaned keys, the CMA `.env.local` URL drift, any Bearer-header fixes), and that Cluster C can now resume. Set this brief's status line to `STATUS: COMPLETE` with the date.

## Estimated total

- Section 1 pre-flight: 5 min
- Section 2 audit: 15-30 min (the Bearer-header audit is the variable — empty result is fast, a hit that needs a code fix adds 15-20 min)
- Section 3 provision: 5 min
- Section 4 cutover: 30-45 min
- Section 5 disable + revoke: 5-10 min
- Section 6 cleanup: 10 min

**Total: 75-105 min of focused work, start to finish.** Realistically ~80 min if the audit is clean (no manual Bearer headers, no missed projects). The leaked key is dead at the end of Section 5.2 — roughly the 55-70 min mark — so the actual exposure closes before cleanup.

## Stop conditions

Stop and ask Brian if:
- The audit surfaces a legacy-key consumer that is hard to redeploy (a non-Vercel host, a cron in another system)
- The Bearer-header audit (2.4) finds manual `Authorization: Bearer <service_role>` usage that needs a non-trivial code change
- The existing `newkey` turns out to be used by a project not in this brief
- A Section 4 verification 401s after an env update (possible key-format mismatch)
- Section 5.2 "Disable JWT-based API keys" errors or won't confirm
- Any production smoke test fails after 5.2

## Rollback plan

**Before Section 5.2:** trivial — re-set the failing project's Vercel env var to the legacy JWT, redeploy. The legacy key is valid until disabled.

**After Section 5.2:** the legacy key is dead, no rollback. Forward fix only — find the missed consumer, set its env var to the new `sb_secret_` key, redeploy. This is why Section 2 audit and Section 4 one-at-a-time verification matter.

## Why future leaks are now a two-click fix

Once HGPG Core is on the new key system, a future leak of an `sb_secret_*` key is handled by: create a new secret key in the dashboard → swap the env var(s) → delete the compromised key. No JWT-secret rotation, no anon-key collateral, no dropped sessions. The remaining friction is purely env-var distribution — addressed by the follow-up below.

## Follow-up (separate sessions, not this one)

- **Other four Supabase projects.** HGPG Listing Reports + MLS, HGPG Signature + Relocation, HGPG FUB Integration, Suna/HGPG1's Project — migrate each to `sb_secret_*` on its own so a leak anywhere is a two-click fix. Not urgent (no known leak); schedule deliberately.
- **Env-var distribution consolidation.** Today each Vercel project stores its own copy of the secret key. Move to a single Vercel team-level shared environment variable so a future rotation is "change one value, redeploy projects" instead of editing N projects. Small task, big payoff — do it right after this migration.
- **Per-app secret keys.** New system allows multiple `sb_secret_*` keys. Eventually give each app its own named secret key so a leak is scoped to one app and the others need no action at all. This is the real end state.
- **Cluster C TM RLS hardening** — was paused on this work; resume in a fresh session after migration. Brief: `projects/cluster-c-path-3-brief.md`.

## Things NOT in scope

- Don't tackle Cluster C in the same session.
- Don't migrate the anon side of any project — HGPG Core anon is already on `sb_publishable_*`; if another project surfaces with legacy anon, document it, don't touch it.
- Don't migrate the other four Supabase projects in this session (see Follow-up).

## Key references

- Brain: https://brain.homegrownpropertygroup.com (web editor) or `~/Documents/hgpg-context` (local clone)
- This brief: `projects/legacy-jwt-rotation.md` in `HGPG1/hgpg-context`
- Supabase project: `ioypqogunwsoucgsnmla` (HGPG Core)
- Supabase API Keys page: https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/api
- Supabase JWT Keys page: https://supabase.com/dashboard/project/ioypqogunwsoucgsnmla/settings/jwt
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- Brain APIs: `https://brain.homegrownpropertygroup.com/api/external/{read,write,commit,log}`
- Cluster C Path 3 brief (next session after migration): `projects/cluster-c-path-3-brief.md`
