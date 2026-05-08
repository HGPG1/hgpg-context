<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Most recent session: 2026-05-08 — CMA wrong-zip candidate fallback shipped 🟢

### What shipped

PR #24 in `HGPG1/hgpg-cma-tool` — `feat(subject-detect): wrong-zip candidate fallback`. Squash-merged as `8bc81a8`. Vercel production deploy READY on `cma.homegrownpropertygroup.com`.

The fully-spec'd queue item from yesterday's handoff is now done end-to-end, including a profile-driven index addition the spec didn't catch.

### What it does

When subject auto-fill's strict (`mls_subject_detect_v2`) and fuzzy (`mls_subject_detect_v2_fuzzy`) paths both miss in the typed ZIP but the same `street_number + street_name` exists under a different ZIP, the lookup now surfaces those candidates through a distinct picker variant ("Did you mean an address in a different ZIP?"). On pick, the form auto-corrects the ZIP field so downstream comp search uses the right postal code.

The 504 Redwine test case (real address in Monroe 28110, agent typed 28210) now confirms instead of falling through to manual entry.

### How it's wired

- New RPC `mls_subject_detect_v2_zip_fallback(p_street_number, p_street_name_tokens)` — drops the `postal_code` filter, returns up to 5 candidates with their actual `postal_code` column. Migration `mls_subject_detect_v2_zip_fallback_2026_05_08` applied to project `wdheejgmrqzqxvgjvfee`.
- Cascade in `lib/mls/subjectDetect.ts` is now: strict → fuzzy → zip-fallback → null. Strict + fuzzy stay can-view-gated; zip-fallback drops both gates because the typed ZIP being wrong is the dominant failure mode.
- `SubjectDetectResult` discriminated union gained `kind: "wrong_zip_candidates"`. `MlsSubjectMatch.context.postalCode` populated only by this path (null on strict/fuzzy).
- `MlsCandidatesPicker.tsx` props changed from `fromFuzzyFallback: boolean` to `variant: "strict" | "fuzzy" | "wrong_zip"`. The wrong_zip variant labels each candidate with its actual ZIP.

### Index gap caught by EXPLAIN ANALYZE — spec was wrong

Yesterday's spec claimed "Same composite index covers it — no new migration if we don't filter by zip." Wrong. `idx_mls_prop_postal_streetnum` leads with `postal_code` and is unusable for a postal-less search. Pre-fix EXPLAIN ANALYZE on a 504 Redwine probe ran **19.6s as a parallel seq scan** over 2.5M rows.

Migration also adds `idx_mls_prop_streetnum` (single-column btree on `street_number`). Post-fix: cold cache ~1s, warm cache **8ms**, well within the 3s `SUBJECT_DETECT_TIMEOUT_MS` budget in `SubjectForm.tsx`.

Lesson reinforces the CLAUDE.md gotcha: **always EXPLAIN ANALYZE before assuming an existing index covers a new lookup.** Even fully-spec'd queue items can ship with the wrong index assumption.

### Other files touched

- `app/seller/new/SubjectForm.tsx` — handles the new kind, auto-corrects ZIP on pick, picker variant prop wired through
- `scripts/test-ranking.ts` — 4 new fixtures (wrong-zip surfaces correct kind/listing/postal, fuzzy hit takes precedence over zip fallback, all-three-empty still returns null)

### Pickup notes for next CMA session

- Smoke-test in production: type "504 Redwine St" with ZIP "28210" on a fresh seller form. Picker should offer the 28110 candidate; on pick, the ZIP field should auto-correct to 28110 and downstream comp search runs in 28110.
- Same-street-neighbor fallback for new construction (Alma Blount type cases) is still queued from yesterday — if the agent types 9644 and the row doesn't exist yet but 9636/9640/9651 do, surface those as a "first home on this block? here are the recent neighbors" hint. Distinct UX from wrong-zip — agent isn't typo'ing, the row is genuinely missing. Defer until a real Alma-Blount-type case shows up live.
- Index audit across other CMA RPCs to confirm none are doing similar full-zip scans is still queued. Same 19.6s pattern could be lurking. Low-priority since `searchComps()` uses the well-indexed `idx_mls_prop_comp_search` path.

---

## Previous session: 2026-05-07 evening to 2026-05-08 morning — CMA subject auto-fill (two-fer) + Claude Code setup 🟢

### Headline wins

1. CMA subject auto-fill diagnosed and fixed twice — index gap + statement_timeout ceiling, two related-but-distinct issues
2. Diagnostic infrastructure shipped (verbose Postgres error format, PR #23 — KEEP this)
3. Claude Code wired up on iMac with full MCP kit (Supabase, Vercel, GitHub) + dangerously-skip-permissions workflow
4. CLAUDE.md ship-it spec dropped into hgpg-cma-tool — next session can be hands-off

---

### CMA Engine: subject auto-fill bug (2-part fix)

**Symptom across both fixes:** subject property auto-fill (the blur handler in the seller form) returning "no match" for valid addresses. Cressingham, Alma Blount, others. Started intermittent, then steady.

**Fix #1 — composite index (2026-05-07 evening):**

Root cause: strict-path RPC `mls_subject_detect_v2` was timing out (Postgres code 57014) because it was doing a bitmap heap scan that read ~20K heap blocks (~160MB) per lookup to find 5 rows. No composite index on `(postal_code, street_number)`.

Migration `add_postal_streetnumber_index_for_subject_detect`:

    CREATE INDEX IF NOT EXISTS idx_mls_prop_postal_streetnum
      ON public.mls_property USING btree (postal_code, street_number)
      WHERE mlg_can_view = true;

Performance: bitmap heap scan (20K buffers, 49ms warm / 8s+ cold) → index scan (36 buffers, 0.4ms execution). ~556x reduction in buffer reads.

**Fix #2 — statement_timeout ceiling (2026-05-08 morning):**

Cressingham started failing again the next morning despite the index. Database EXPLAIN ANALYZE showed query running in 0.4ms. So the database was fast but the lambdas were still hitting `code=57014`.

Root cause: pooler/network latency + cold-lambda connection setup were occasionally pushing total operation time over the default 8s `statement_timeout` for the `authenticated` role. Postgres counts connection-acquisition wait time toward statement_timeout, not just query execution.

Migration `raise_authenticated_statement_timeout_for_subject_detect`:

    ALTER ROLE authenticated SET statement_timeout = '30s';

Defensive ceiling, not a target. Real queries still run instant. Affects all `authenticated` queries — fine because nothing user-facing should ever run anywhere near 30s. If we ever see a real 20s+ query that's a bug surfaced, not hidden.

**Both fixes were necessary, neither replaces the other.** Index makes the query fast. Higher timeout absorbs network/pooler variance.

### Diagnostic patch (PR #23) — KEEP

`HGPG1/hgpg-cma-tool` PR #23 — `Surface full Postgres error on subject-detect RPC failure` — squash-merged.

Two error-throw lines in `lib/mls/subjectDetect.ts` (327, 353) now include `code`, `details`, `hint` from the Supabase error object alongside `.message`. Without this we never would have caught either timeout — the original `${err.message}` truncated in Vercel's table view to `[subjectDetect] auto-fill f...`. With the patch live we got `code=57014` immediately for both diagnoses.

This patch paid for itself within an hour. **DO NOT remove it during future "cleanup."**

### Bugs found that are NOT bugs (data gaps, not code bugs)

- **9644 Alma Blount Blvd 28277**: address genuinely not in MLS data. Neighbors at 9636, 9640, 9651, 9652, 9728 in DB. 9644 specifically was never on Carolina MLS or pre-dates MLS Grid backfill. Tool correctly returns no match.
- **504 Redwine St (zip mistyped)**: real address but in Monroe 28110, not the zip Brian typed. Tool correctly returns no match because strict RPC filters on exact `postal_code`.

### Pickup queued — wrong-zip candidate fallback (in-flight)

When subject auto-fill returns null, the form shows "No MLS match found, fill in manually." Correct but unhelpful when the underlying cause is a wrong zip.

Spec: new RPC `mls_subject_detect_v2_zip_fallback(p_street_number, p_street_name_tokens)` returning up to 5 rows matching street_number + street_name across all zips. New `kind: "wrong_zip_candidates"` in `SubjectDetectResult`. Distinct UI in `MlsCandidatesPicker` ("Did you mean an address in a different ZIP?"). Same composite index covers it.

**Status as of this writeup: Claude Code is mid-flight on this in `~/Documents/hgpg-cma-tool` with `--dangerously-skip-permissions`. PR may already be merged by the time you read this — check GitHub.**

Also pickup-worthy:
- Same-street-neighbor fallback for new construction (Alma Blount type cases)
- Index audit across other CMA RPCs to confirm none are doing similar full-zip scans

---

### Claude Code setup (iMac complete, Mac mini pending)

Solution to the babysit-the-paste-loop dynamic during web Claude debug sessions: get Claude Code's MCP kit equivalent to web Claude's so future sessions ship end-to-end without Brian in the middle.

**iMac (Home Office) — DONE 2026-05-07:**

- claude.ai Supabase ✓ (managed connector)
- claude.ai Vercel ✓ (via plugin path)
- claude.ai Gmail ✓
- claude.ai Google Calendar ✓
- claude.ai Google Drive ✓
- claude.ai Canva ✓
- github ✓ (local install, fine-grained PAT, scope=user)
- claude.ai Slack — skipped, not actively using

**GitHub PAT (iMac):**
- Token name: `Claude Code MCP - iMac`
- Scopes: HGPG1 org, Contents R/W, Pull requests R/W, Issues R/W, Metadata R, Actions R, Workflows R/W
- Stored in `~/.claude.json` (plaintext, file perms protect it)
- Expiration set on creation — calendar reminder needed to rotate

**Mac mini — TODO next time Brian works there:**

Repeat:

    claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/

Generate a separate PAT named `Claude Code MCP - Mac mini` so they can be revoked independently.

The claude.ai connectors auto-sync across machines on first Claude Code login — only GitHub needs per-machine PAT.

**Hands-off workflow:**

    cd ~/Documents/<repo>
    git pull
    claude --dangerously-skip-permissions

Or alias:

    alias claudego="claude --dangerously-skip-permissions"

Then `claudego` in any HGPG repo. The flag skips per-command permission prompts so Claude Code can branch / patch / push / merge without you saying yes 40 times. Anthropic-level safety guardrails still apply, and CLAUDE.md "stop and ask" rules still trigger for risky work.

### CLAUDE.md ship-it spec — hgpg-cma-tool

Dropped into `hgpg-cma-tool/main` (commit `300018a`). Tells Claude Code:

- Read CONTEXT.md + SESSION-HANDOFF.md from brain at session start
- Cook freely on diagnosed bugs (branch, patch, push, PR, squash-merge with `--auto`)
- Stop and ask only for: 5+ file changes, schema migrations, RLS/auth changes, `compSearch.ts` changes, env var changes, or refactors
- Conventions: commit author `brian@homegrownpropertygroup.com`, `/* Last Updated */` on touched files, no em dashes, snake_case for Supabase columns
- CMA-specific gotchas baked in (unparsed_address null, GLA rate, PIN auth, etc.)

**Next session worth doing:** drop equivalent CLAUDE.md files into the other HGPG repos in priority order: `hgpg-transaction-manager`, `brain-app`, `charlotte-new-construction-nextjs`, `south-charlotte-report`, `homegrown-property-group-site`. Each calibrated to that repo's quirks.

---

### Pickup checklist for next session

1. Verify the wrong-zip-fallback PR landed (Claude Code in-flight). If yes, smoke test on a real wrong-zip address. If no, debug whatever stopped it.
2. Decide which repo gets the next CLAUDE.md (TM is highest impact).
3. Repeat MCP + GitHub PAT setup on Mac mini if/when next session starts there.

---

## Previous session: 2026-05-06 — Brain App MVP shipped 🟢

### What got built
- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- GitHub auth: fine-grained PAT scoped to `HGPG1/hgpg-context`, contents:write only
- Round-trip verified: edit file in browser, commit lands on `main` with author `brian@homegrownpropertygroup.com`

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
- Magic link redirected to `tools.homegrownpropertygroup.com` (Supabase Site URL fallback), fixed by adding `/auth/callback` route handler that was missing from initial scaffold + pointing `emailRedirectTo` at it
- Supabase free SMTP rate limit (2/hr) hit during testing, fixed permanently by switching to Resend custom SMTP

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

### Pickup notes
- Brain-app is live and working, use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
- The `package-lock.json` may differ between iMac and Mac mini, push from whichever machine you most recently ran `npm install` on
