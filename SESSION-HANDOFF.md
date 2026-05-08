<!-- Last Updated: 2026-05-07 -->

# Session Handoff

## Last session: 2026-05-07 — CMA subject auto-fill fixed + Claude Code setup 🟢

### Headline wins

1. CMA subject auto-fill bug diagnosed and fixed (RPC timeout, missing index)
2. Diagnostic infrastructure shipped (verbose Postgres error format, PR #23)
3. Claude Code wired up on iMac with full MCP kit (Supabase, Vercel, GitHub)
4. CLAUDE.md ship-it spec dropped into hgpg-cma-tool — next CMA session can be hands-off

---

### CMA Engine: subject auto-fill bug

**Symptom:** subject property auto-fill (the blur handler in the CMA tool's seller form) was returning "no match" for valid addresses. Started Brian fearing a real bug; field was failing on Cressingham, Alma Blount, and others.

**Root cause:** the strict-path RPC `mls_subject_detect_v2` was timing out (Postgres error code `57014`, `canceling statement due to statement timeout`) on cold buffer cache. Function was doing a bitmap heap scan that read ~20K heap blocks (~160MB) per lookup to find 5 rows — no composite index on `(postal_code, street_number)`.

**Why it presented as "intermittent":** warm cache made it work in 49ms. Cold cache (post-MLS-sync flush, post-cold-lambda-start, post-deploy) blew past the 8-second statement timeout. Looked random; was actually deterministic.

**Fix applied:** migration `add_postal_streetnumber_index_for_subject_detect` on Supabase project `wdheejgmrqzqxvgjvfee`:

    CREATE INDEX IF NOT EXISTS idx_mls_prop_postal_streetnum
      ON public.mls_property USING btree (postal_code, street_number)
      WHERE mlg_can_view = true;

**Performance:**
- Before: 49ms warm / 8s+ cold (timeout)
- After: 2.4ms warm / ~50-100ms cold expected
- Plan: bitmap heap scan to index scan, 20,025 buffers to 36 buffers (~556x reduction)

No code changes needed for the fix itself. Brian verified 5022 Cressingham Dr 29707 works after the migration.

### Diagnostic patch (PR #23)

`HGPG1/hgpg-cma-tool` PR #23 — `Surface full Postgres error on subject-detect RPC failure` — squash-merged.

Changed two error-throw lines in `lib/mls/subjectDetect.ts` (327, 353) from `${err.message}` to a fuller string that includes `code`, `details`, `hint`. Without this we would never have caught the 57014 timeout — the original `${err.message}` was getting truncated in Vercel's table view to `[subjectDetect] auto-fill f...`. With the patch live, we got the full `code=57014` text immediately.

**Keep this patch.** It's diagnostic infrastructure that paid for itself within an hour. If anyone proposes "cleaning it up" in the future, push back hard.

### Bugs found that are NOT bugs

- **9644 Alma Blount Blvd 28277:** address genuinely not in MLS data. Neighbors at 9636, 9640, 9651, 9652, 9728 all in DB; 9644 specifically was never on Carolina MLS or pre-dates MLS Grid backfill. Tool correctly returns no match.
- **504 Redwine St (zip mistyped):** real address but in Monroe 28110, not the zip Brian typed. Tool correctly returns no match because strict RPC filters on exact `postal_code`.

### Pickup notes for next CMA session — wrong-zip candidate fallback

Real UX gap: when subject auto-fill returns null, the form shows "No MLS match found, fill in manually." Correct but unhelpful when the underlying cause is a wrong zip (Brian fat-fingered Redwine and got nothing, no hint why).

Proposed fix (fully spec'd, ready to ship):

1. Create new RPC `mls_subject_detect_v2_zip_fallback(p_street_number, p_street_name_tokens)` — searches by `street_number + street_name` ignoring zip, returns up to 5 rows ordered by `modification_timestamp DESC`. Same RETURNS TABLE shape as v2/v2_fuzzy + a new `postal_code` column so the picker can show "Monroe 28110" next to each candidate.
2. Wire into `createSupabaseSubjectLookup` as a third call after fuzzy returns empty. Add `zipFallbackRows` to `SubjectMlsLookup`'s return type.
3. Add new `kind` to `SubjectDetectResult` — `kind: "wrong_zip_candidates"`. Render distinct UI in `MlsCandidatesPicker` ("Did you mean an address in a different ZIP?").
4. Same composite index covers it — no new migration if we don't filter by zip.

Estimated effort: 30-45 min including tsc + deploy. Send Brian into Claude Code in `~/Documents/hgpg-cma-tool` and have it cook end-to-end.

Also pickup-worthy:
- Same-street-neighbor fallback for new construction (Alma Blount type cases)
- Index audit across other CMA RPCs to confirm none are doing similar full-zip scans

---

### Claude Code setup (iMac)

Brian hit a wall with the babysit-the-paste-loop dynamic during today's CMA debug. Solution: get Claude Code's MCP kit equivalent to web Claude's so future sessions can ship end-to-end without him in the middle.

**Installed on iMac (Home Office):**

- claude.ai Supabase ✓ (managed connector)
- claude.ai Vercel ✓ (via plugin path)
- claude.ai Gmail ✓
- claude.ai Google Calendar ✓
- claude.ai Google Drive ✓
- claude.ai Canva ✓
- github ✓ (local install, fine-grained PAT, scope=user)
- claude.ai Slack — skipped, not actively using

**GitHub PAT details:**
- Token name: `Claude Code MCP - iMac`
- Scopes: HGPG1 org, Contents R/W, Pull requests R/W, Issues R/W, Metadata R, Actions R, Workflows R/W
- Stored in `~/.claude.json` (plaintext, file perms protect it)
- Expiration set on creation — calendar reminder needed to rotate

**Important — needs same setup on Mac mini:**
The Mac mini hasn't been MCPed yet. Repeat this on the Mac mini next time Brian works there:

    claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/

Generate a separate PAT named `Claude Code MCP - Mac mini` so they can be revoked independently.

The claude.ai connectors (Supabase, Vercel, etc.) auto-sync across machines on first Claude Code login — no per-machine setup needed for those. Only GitHub needs the per-machine PAT.

### CLAUDE.md ship-it spec

Dropped a CLAUDE.md into `hgpg-cma-tool/main` (commit `300018a`). Tells Claude Code:

- Read CONTEXT.md + SESSION-HANDOFF.md from brain at session start
- Cook freely on diagnosed bugs (branch, patch, push, PR, squash-merge with `--auto`)
- Stop and ask only for: 5+ file changes, schema migrations, RLS/auth changes, `compSearch.ts` changes, env var changes, or refactors
- Conventions: commit author `brian@homegrownpropertygroup.com`, `/* Last Updated */` on touched files, no em dashes, snake_case for Supabase columns
- CMA-specific gotchas baked in (unparsed_address null, GLA rate, PIN auth, etc.)

**Next session worth doing:** drop equivalent CLAUDE.md files into the other HGPG repos in priority order: `hgpg-transaction-manager`, `brain-app`, `charlotte-new-construction-nextjs`, `south-charlotte-report`, `homegrown-property-group-site`. Each one calibrated to that repo's quirks.

---

### Project status updates

- `projects/cma-engine.md` — note the timeout fix + diagnostic patch + CLAUDE.md. Subject auto-fill now reliable.
- `projects/transaction-manager.md` — no changes. CLAUDE.md to be added next session.
- `projects/brain-app.md` — no changes today.

### Pickup checklist for next session

1. Decide which repo gets the next CLAUDE.md (TM is highest impact)
2. Either ship the wrong-zip-fallback CMA improvement directly via Claude Code in the cma-tool repo, OR queue it for whenever
3. Repeat MCP + GitHub PAT setup on Mac mini if/when next session starts there

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
