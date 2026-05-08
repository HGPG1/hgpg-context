<!-- Last Updated: 2026-05-08 -->

# Session Handoff

## Most recent session: 2026-05-08 — CMA adjustment engine bugs found + queued 🟡

### State as Brian leaves the desk

- Claude Code in-flight on the wrong-zip-fallback fix in `~/Documents/hgpg-cma-tool` with `--dangerously-skip-permissions`. Was actively working at last check. May or may not be done by next session — verify in GitHub PRs.
- 6 NEW bugs identified in the CMA adjustment engine after Brian ran 4 production CMAs (215 Tyndale Ct, 504 Redwine St, 5601 Medlin Rd, 6022 Candlestick Ln). Full diagnosis written to `projects/cma-engine-bugs-2026-05-08.md` in the brain. Claude Code prompt for fixes prepared and saved (in chat).
- **Brian is hitting the road.** Resume by reading the bugs file, checking the in-flight PR, then deciding when to fire the bugs prompt at a fresh Claude Code session.

### CRITICAL — do not send CMAs to clients

The 4 reports run today have real adjustment math bugs. Numbers are wrong by enough to kill financing on a deal. Until the engine fixes ship, manually review every CMA against the comp set before sending — especially the feature parity and GLA-vs-basement adjustments.

Bugs in priority order (full detail in `projects/cma-engine-bugs-2026-05-08.md`):

1. **Bug 5 — confidence bounds collapse to single value** (CRITICAL math bug). 504 Redwine and 5601 Medlin shipped with Low == High, Medlin had PMV anchor *above* both bounds.
2. **Bug 1 — feature parity broken** (CRITICAL). The "Subject features" adjustment adds full +$70K to every comp regardless of whether the comp has those features. MLS remarks for 7026 Wyngate explicitly state "finished walk out basement" — engine still applied subject-only $25K basement adjustment.
3. **Bug 2 — GLA includes basement sqft** (CRITICAL, appraisal violation). Subject sqft is matched 1:1 against comp sqft at $60/sqft, ignoring URAR Form 1004 standard that GLA = above-grade only. Basement should be separate at $25-30/sqft finished, $10/sqft unfinished. 6022 Candlestick subject reported as 5,055 sqft total (basement included) which is appraisal-incorrect.
4. **Bug 6 — Active comp weighting too generous when Sold count is thin**. Currently 5 Actives × 0.08 = 0.40 collective weight. Anchor gets pulled toward asking prices.
5. **Bug 4 — PMV anchor sanity check missing**. Candlestick anchor was $1.172M, but no individual comp adjusted to anywhere near that — synthetic average of two distant clusters.
6. **Bug 3 — outlier penalty fires asymmetrically on near-twin comps**.

### What was actually shipped today

- Migration `add_postal_streetnumber_index_for_subject_detect` (yesterday evening): composite index on `mls_property(postal_code, street_number)`. Subject auto-fill query: 49ms warm / 8s+ cold → 2.4ms warm / ~50-100ms cold expected.
- Migration `raise_authenticated_statement_timeout_for_subject_detect` (this morning): bumped statement_timeout for `authenticated` role from 8s to 30s. Defensive ceiling for cold lambda + pooler latency.
- PR #23 (yesterday): `Surface full Postgres error on subject-detect RPC failure` — verbose error format in `lib/mls/subjectDetect.ts`. Diagnostic infrastructure. **DO NOT remove this.**
- CLAUDE.md ship-it spec dropped into `hgpg-cma-tool/main` (commit `300018a`).
- Claude Code MCP setup on iMac (Home Office): Supabase, Vercel, GitHub, Gmail, Calendar, Drive, Canva all connected. Slack skipped. Mac mini still needs setup next time Brian works there.
- Brain bug spec written to `projects/cma-engine-bugs-2026-05-08.md` (commit `47787eb`).
- Wrong-zip-fallback (in-flight): Claude Code is shipping it now, not yet verified.

### Pickup checklist for next session

1. **Check GitHub PRs in HGPG1/hgpg-cma-tool** — did the wrong-zip-fallback land? If yes, smoke test on a real wrong-zip address. If stuck, debug whatever stopped Code.
2. **Read `projects/cma-engine-bugs-2026-05-08.md`** — full diagnosis of the 6 bugs.
3. **Fire the CMA bugs Claude Code prompt** (saved in chat) at a fresh Claude Code session in `~/Documents/hgpg-cma-tool` with `--dangerously-skip-permissions`. Prompt has explicit pause points after PR 3 and PR 4 — Brian should be at the keyboard for those checkpoints because PR 5 involves a Supabase schema change that needs review.
4. **Mac mini Claude Code MCP setup** if/when next session starts there. Repeat: `claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/`. Generate separate PAT named "Claude Code MCP - Mac mini".
5. **Eventually decide** which other repos get CLAUDE.md ship-it specs. TM is highest impact next.

### Test reports for regression testing the bug fixes

- `8023175d-37c8-45d7-95da-b9245abe4761` (6022 Candlestick Ln, Lancaster) — feature parity bug, basement-in-GLA bug, outlier asymmetry
- 504 Redwine and 5601 Medlin reports — pull IDs from `cma_reports` by subject_address before fixing Bug 5

After fixes, expected behavior on Candlestick:
- 7026 Wyngate adjusted price drops from $991,500 to ~$925-940K (basement parity, primary-on-main parity)
- 5124 Mill Race weight drops to <0.5 due to far + pool + submarket mismatch
- PMV anchor lands $880K-$960K, NOT $1.17M
- Aspirational/PMV/Event spread tightens accordingly

### CMA subject auto-fill (RESOLVED 2026-05-08)

Two-part fix that landed yesterday evening through this morning:

**Part 1 — composite index.** Original symptom: subject auto-fill returning "no match" for valid addresses. Strict-path RPC `mls_subject_detect_v2` was timing out (Postgres code 57014) on cold buffer cache. Bitmap heap scan reading ~20K heap blocks per lookup. Fixed with `CREATE INDEX idx_mls_prop_postal_streetnum ON mls_property(postal_code, street_number) WHERE mlg_can_view = true`.

**Part 2 — statement_timeout ceiling.** Cressingham started failing again next morning despite the index. Database EXPLAIN ANALYZE showed query running in 0.4ms but lambdas still hitting code=57014. Pooler/network latency + cold-lambda connection setup were occasionally pushing total operation time over 8s. Bumped `authenticated` role statement_timeout to 30s. Real queries still run instant. If we ever see a real 20s+ query that's a bug surfaced, not hidden.

**Both fixes were necessary, neither replaces the other.** Index makes the query fast. Higher timeout absorbs network/pooler variance.

### Diagnostic patch (PR #23) — KEEP

Two error-throw lines in `lib/mls/subjectDetect.ts` (327, 353) now include `code`, `details`, `hint` from the Supabase error object alongside `.message`. Without this we never would have caught either timeout. **DO NOT remove during future "cleanup."**

### Bugs found that are NOT bugs (data gaps, not code bugs)

- **9644 Alma Blount Blvd 28277**: address genuinely not in MLS data. Tool correctly returns no match.
- **504 Redwine St (zip mistyped)**: real address but in Monroe 28110, not the zip Brian typed. Tool correctly returns no match.

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

### Brain App write API (added 2026-05-07 evening)
- POST `https://brain.homegrownpropertygroup.com/api/external/write`
- Authorization: Bearer token (in 1Password as "HGPG Brain Write Token")
- JSON body: `{ path, content, message?, branch? }`
- Authored as brian@homegrownpropertygroup.com
- 1MB content cap. Blocked paths: `.git/`, `.github/workflows/`, `.vercel/`, `node_modules/`, `.env*`, `package-lock.json`
- Used by web Claude sessions to update brain files without manual paste

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

### Pickup notes
- Brain-app is live and working, use it for any future updates to `hgpg-context`
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev` on Mac mini (work machine)
- Brain-app local on iMac: same setup, repo at `~/Developer/brain-app` if rebuilt, otherwise needs fresh `gh repo clone HGPG1/brain-app` + `npm install` + `cp env.example .env.local`
