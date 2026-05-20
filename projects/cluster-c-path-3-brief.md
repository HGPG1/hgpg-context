<!-- Last Updated: 2026-05-20 -->

# Claude Code Brief: TM RLS Hardening (Cluster C, Path 3)

## Goal

Close the public-anon-key write hole on 17 tables in HGPG Core (Supabase project `ioypqogunwsoucgsnmla`) without breaking TM or CMA Engine.

Today, the `anon` role has full SELECT/INSERT/UPDATE/DELETE on every table below, gated only by `USING(true)` RLS policies — which means anyone with the published `NEXT_PUBLIC_SUPABASE_ANON_KEY` can read or modify the entire database via PostgREST.

After this work: anon role has no access to these tables. App code routes writes through service-role helpers. End-user reads either go through dedicated API routes (already service-role) or use Google-OAuth-cookie-gated routes.

This is the 80/20 fix from the May 20 Supabase security audit. The audit deferred this as "needs a focused session" — that session is now.

## Scope

**Repos involved:**
- `HGPG1/hgpg-transaction-manager` (primary — Next.js, deployed at closings.homegrownpropertygroup.com)
- `HGPG1/hgpg-cma-tool` (secondary — Next.js, deployed at cma.homegrownpropertygroup.com)

**Supabase project:** `ioypqogunwsoucgsnmla` (HGPG Core)

**Tables in scope (17):**
- TM core: `transactions`, `transaction_milestones`, `transaction_parties`, `transaction_tasks`, `transaction_terms`, `transaction_documents`, `transaction_activity`, `transaction_deadlines`
- TM templates: `milestone_templates`, `milestone_audit_log`, `document_templates`, `task_templates`, `text_templates`, `reminder_templates`
- TM ReZEN: `rezen_review_candidates`
- CMA: `cma_reports`, `cma_adjustment_defaults`

## Why this is safe to attempt

1. TM already has Google OAuth + cookie sessions (`lib/auth.ts`) — all UI traffic is already authenticated at the app layer.
2. TM already has a service-role helper: `getServiceClient()` in `lib/createTransaction.ts` (uses `SUPABASE_SERVICE_ROLE_KEY`). It just isn't used everywhere it should be.
3. Apps share a Supabase project — both TM and CMA Engine use HGPG Core — so a single policy migration covers both.
4. Service role bypasses RLS, so any route that already uses `getServiceClient()` will keep working unchanged after we revoke anon grants.

## Why this needs care

1. Most TM API routes use the anon `supabase` import from `lib/supabase.ts`. Every one of those will start failing once we revoke anon grants. Each needs to be either:
   - Switched to `getServiceClient()` (server-side admin routes — most TM routes)
   - Gated by Google OAuth cookie + still uses service-role to query (the right pattern)
2. CMA Engine has its own anon client (`lib/supabase.ts` reads `SUPABASE_ANON_KEY`). It has no service-role helper today — one needs to be added.
3. tc-concierge runs embedded in TM at `/intake` — its API routes also need auditing.
4. The Supabase tracker token endpoint (`/track/:token`) uses anon to call a SECURITY DEFINER function (`get_tracker_by_token`). That keeps working because SECURITY DEFINER bypasses RLS by design. Don't break that.

## Required workflow (read before starting)

1. **Brain-first.** Pull `CONTEXT.md` and `SESSION-HANDOFF.md` from raw.githubusercontent.com/HGPG1/hgpg-context/main/ first. The May 20 session entry has full advisor context and the exact current policy state.

2. **Diagnostic-first.** Don't write the policy migration before you've grepped every API route. The migration is the last step, not the first.

3. **Verify with Supabase MCP** if it's configured. Otherwise use `supabase db` CLI or the dashboard SQL editor for the policy migration step.

## Concrete steps

### Step 1: Audit TM API routes (30-45 min)

```bash
cd ~/Documents/HGPG/hgpg-transaction-manager  # adjust path
# Find every file importing the anon supabase client
grep -rn "from ['\"].*lib/supabase['\"]" app/ lib/ --include="*.ts" --include="*.tsx"
# Find every file using getServiceClient
grep -rn "getServiceClient" app/ lib/ --include="*.ts" --include="*.tsx"
# Find every Supabase .from() call - these are the queries that need to route through service role
grep -rn "supabase\.from\|sb\.from" app/api/ --include="*.ts"
```

Build a list: route → which Supabase client it uses → which tables it touches → whether it needs auth.

Most TM routes will be admin routes (TC/agent doing work in the UI) — they should all use service role + Google OAuth cookie check. The cookie check (`requireUser()` in `lib/auth.ts`) is the right gate; RLS is not.

### Step 2: Audit CMA Engine routes (15-30 min)

```bash
cd ~/Documents/HGPG/hgpg-cma-tool
grep -rn "from ['\"].*lib/supabase['\"]" app/ lib/ --include="*.ts" --include="*.tsx"
grep -rn "supabase\.from\|getSupabase" app/api/ --include="*.ts"
```

CMA Engine has no service-role helper today. You'll need to add one. Pattern:

```typescript
// lib/supabaseAdmin.ts
import { createClient, type SupabaseClient } from "@supabase/supabase-js";

let _client: SupabaseClient | null = null;

export function getServiceClient(): SupabaseClient {
  if (_client) return _client;
  const url = process.env.SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error("SUPABASE_URL or SUPABASE_SERVICE_ROLE_KEY missing");
  _client = createClient(url, key, { auth: { persistSession: false } });
  return _client;
}
```

Then add `SUPABASE_SERVICE_ROLE_KEY` to Vercel env for the CMA Engine project (`prj_...`, ask Brian if you don't know the project ID — it's documented in the brain at `projects/cma-engine.md`).

### Step 3: Migrate routes one folder at a time (2-3 hours)

For each TM route that touches one of the 17 tables:
- If it's a server-side admin operation (e.g. POST /api/transactions/create, PATCH /api/milestones/[id]) — switch the import from `lib/supabase` to `lib/createTransaction` and use `getServiceClient()`.
- If it's a public route that needs anon access (e.g. the tracker endpoint), confirm it's calling a SECURITY DEFINER function (not a raw `.from(table)`). If it isn't, refactor.

**Commit in small batches.** Don't rewrite all 60 routes in one commit. Group by table:
- Batch 1: all routes touching `transactions` + `transaction_milestones`
- Batch 2: all routes touching `transaction_parties` + `transaction_documents`
- ...etc

Each batch should pass a manual TM smoke test before moving to the next.

### Step 4: Run the policy migration (5 min)

After every route is verified to use service-role or SECURITY DEFINER, apply this migration via Supabase MCP `apply_migration` (or the dashboard SQL editor if MCP isn't available):

```sql
-- Drop all the allow_all policies
DROP POLICY IF EXISTS "allow_all" ON public.transactions;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_milestones;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_parties;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_tasks;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_terms;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_documents;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_activity;
DROP POLICY IF EXISTS "allow_all" ON public.transaction_deadlines;
DROP POLICY IF EXISTS "milestones_auth_delete" ON public.transaction_milestones;
DROP POLICY IF EXISTS "milestones_auth_insert" ON public.transaction_milestones;
DROP POLICY IF EXISTS "milestones_auth_select" ON public.transaction_milestones;
DROP POLICY IF EXISTS "milestones_auth_update" ON public.transaction_milestones;
DROP POLICY IF EXISTS "milestone_audit_auth_insert" ON public.milestone_audit_log;
DROP POLICY IF EXISTS "milestone_audit_auth_select" ON public.milestone_audit_log;
DROP POLICY IF EXISTS "milestone_templates_auth_delete" ON public.milestone_templates;
DROP POLICY IF EXISTS "milestone_templates_auth_insert" ON public.milestone_templates;
DROP POLICY IF EXISTS "milestone_templates_auth_select" ON public.milestone_templates;
DROP POLICY IF EXISTS "milestone_templates_auth_update" ON public.milestone_templates;
DROP POLICY IF EXISTS "allow_all" ON public.document_templates;
DROP POLICY IF EXISTS "allow_all" ON public.task_templates;
DROP POLICY IF EXISTS "Allow all for service role" ON public.text_templates;
DROP POLICY IF EXISTS "Allow read access" ON public.reminder_templates;
DROP POLICY IF EXISTS "Allow service role update" ON public.reminder_templates;
DROP POLICY IF EXISTS "service role full access" ON public.rezen_review_candidates;
DROP POLICY IF EXISTS "cma_reports_open_all" ON public.cma_reports;
DROP POLICY IF EXISTS "cma_adjustment_defaults_open_all" ON public.cma_adjustment_defaults;

-- Revoke anon and authenticated grants
REVOKE ALL ON public.transactions FROM anon, authenticated;
REVOKE ALL ON public.transaction_milestones FROM anon, authenticated;
REVOKE ALL ON public.transaction_parties FROM anon, authenticated;
REVOKE ALL ON public.transaction_tasks FROM anon, authenticated;
REVOKE ALL ON public.transaction_terms FROM anon, authenticated;
REVOKE ALL ON public.transaction_documents FROM anon, authenticated;
REVOKE ALL ON public.transaction_activity FROM anon, authenticated;
REVOKE ALL ON public.transaction_deadlines FROM anon, authenticated;
REVOKE ALL ON public.milestone_templates FROM anon, authenticated;
REVOKE ALL ON public.milestone_audit_log FROM anon, authenticated;
REVOKE ALL ON public.document_templates FROM anon, authenticated;
REVOKE ALL ON public.task_templates FROM anon, authenticated;
REVOKE ALL ON public.text_templates FROM anon, authenticated;
REVOKE ALL ON public.reminder_templates FROM anon, authenticated;
REVOKE ALL ON public.rezen_review_candidates FROM anon, authenticated;
REVOKE ALL ON public.cma_reports FROM anon, authenticated;
REVOKE ALL ON public.cma_adjustment_defaults FROM anon, authenticated;
```

This drops the policies and revokes the grants in one transaction. After this, only the service role can touch these tables. RLS stays enabled (deny-all without policies).

### Step 5: Smoke test (20 min)

In production after the migration:
1. Load TM dashboard — should show deals.
2. Open any active deal — milestones/parties/tasks/documents should all render.
3. Edit a milestone (change date or status) — should save.
4. Add a party to a deal — should save.
5. Open CMA Engine at cma.homegrownpropertygroup.com, generate a CMA on any test address — should produce a packet.
6. Trigger a tracker URL (`/track/:some_token`) — should still work (SECURITY DEFINER path).
7. Check Supabase advisor — should show zero Cluster C lints.

If anything 401s or returns empty data, the failure is in a route you missed in Step 3 — find it, switch it to service role, redeploy.

### Step 6: Update brain (10 min)

Append a new entry to the top of `SESSION-HANDOFF.md` in `HGPG1/hgpg-context` via the brain write API (`POST /api/external/write`). Mark Cluster C resolved. Note any routes you migrated for future reference.

## Estimated total: 4-6 hours

- Audit: 1 hour
- Service-role helper in CMA: 15 min
- Route migration: 2-3 hours
- Migration + smoke test: 30 min
- Brain update: 10 min

## Rollback plan

If anything goes catastrophically wrong after Step 4:

```sql
-- Re-create the most permissive policy as a hotfix
CREATE POLICY "allow_all_temp" ON public.transactions FOR ALL USING (true) WITH CHECK (true);
GRANT ALL ON public.transactions TO authenticated;
-- ...repeat for whichever tables are 401ing
```

This restores function in 30 seconds while you debug. Then you have time to find the missed route and re-apply the lockdown.

## Things NOT in scope

- Don't refactor TM auth itself. Don't move TM to Supabase Auth — that's a bigger project. Cookie-based Google OAuth + service-role queries is fine.
- Don't touch `get_tracker_by_token` — that's intentionally SECURITY DEFINER and intentionally callable by anon.
- Don't touch the 24 tables that already had anon grants revoked in the May 20 session.
- Don't tackle Cluster D (get_tracker_by_token advisor warnings) or Cluster E (pg_trgm) — both are deferred indefinitely as known noise.

## Key references

- Brain: https://brain.homegrownpropertygroup.com (web editor) or `~/Documents/hgpg-context` (local repo)
- Project state: `projects/transaction-manager.md`, `projects/cma-engine.md`
- Last session: `SESSION-HANDOFF.md` top entry (2026-05-20 Supabase security cleanup)
- Bearer token for brain APIs: in Brian's keychain as BRAIN_WRITE_TOKEN
- Supabase project ID: `ioypqogunwsoucgsnmla` (HGPG Core)
- Vercel team: `team_FietQPKCmnyioG2n0FdteQCV`
- TM Vercel project: documented in brain `infrastructure.md`
- CMA Vercel project: `prj_oLWVcE4J1UKzJtmggoQCOW35LUhy`

## Stop conditions

Stop and ask Brian if:
- The route audit turns up more than 5 routes that don't have an obvious Google-OAuth cookie gate (i.e. they might be intentionally public-but-anon — the tracker route is one such case).
- You find any route that writes to one of the 17 tables AND is intentionally hit by external systems (webhooks, integrations) without a cookie. Those need their own auth gate (a shared secret bearer token, e.g.).
- The migration in Step 4 errors on any DROP POLICY or REVOKE — could mean schema has drifted since this brief was written.
- A smoke test fails in a way that doesn't reduce to "missed route uses anon client" — i.e. anything weirder than a 401.
