<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Most recent session: 2026-05-11 — CMA engine math sweep complete + autosave + appreciation tuning 🟢

### State as Brian closes the day

Engine bug queue from the original 13-bug sweep is fully drained. Autosave shipped. Time-of-sale appreciation calibrated to actual Charlotte metro 2024-2025 cooling market via real-world Cressingham verification. Tyndale verified with model-id hotfix.

`cma.homegrownpropertygroup.com` is in a materially different place than 24 hours ago. Three days of bug work (2026-05-07 through 2026-05-11) closed.

### CRITICAL — Open mobile UI bug surfaced 2026-05-11 evening

Brian observed on iOS Safari: the global header ("Seller Buyer Appraiser History" nav) overflows past viewport on mobile widths. /appraiser/new page heading and intro paragraph have no left/right gutter (flush to left edge). Form fields below have padding but the page wrapper does not. Result: horizontal page scroll and a header that visually clips at the right edge.

Math-engine and packet generation are unaffected. This is pure CSS / responsive layout.

Likely fix: viewport meta tag confirmation + page wrapper max-width/overflow + responsive header (hamburger or wrap below 600px). 15-20 min total. NOT a blocker for desktop agent workflow. Capture as "Enhancement 2 — mobile responsive pass" for next session.

### Final verification — Cressingham 5022 post-Bug-14

PMV landed at **$670,983** with MODERATE confidence band $646,540–$714,160. Five Solds + one active, no exclusions, tight cluster. Confidence label correctly upgraded from "low" to "moderate."

Per-comp narratives now correctly attribute time-of-sale lines (no more "newer vintage" hallucinations). The LLM ATTRIBUTION RULES added in PR #36 prevent the model from labeling adjustments with categories the engine didn't compute.

Cressingham journey across three days:
- May 8 morning: $659K (pre-PR-35, no time-of-sale)
- May 8 post-PR-35: $704K (5% appreciation default too aggressive)
- May 11 post-PR-36: $670,983 (3% default calibrated to real market)

### Final verification — Tyndale 215 post-model-id-hotfix

PMV $788K, Aspirational $818K, Event $767K. Tight ~4% strategy spread, appropriate for Somerset 28277 hot market. Narratives correctly engage with market context ("median DOM is 2 days," "relocating executive or out-of-market household that hasn't yet internalized local price anchors"). Bug 11 strategy band cap working — Aspirational properly capped inside High vs yesterday's overshoot.

Initial Tyndale attempt failed with "Generation failed — Load failed" client error. Vercel logs showed /api/packet/seller returning 200 with warning "The model 'claude-sonnet-4-...". Diagnosis: deprecated model snapshot ID. Code shipped a hotfix updating to current Sonnet, retests clean. Logs at 02:00 ET show no errors.

### PRs shipped today (2026-05-11)

| PR | Bugs | Description |
|----|------|-------------|
| #34 | Enhancement 1 | Autosave on /seller/adjust with editVersion counter + 500ms debounce. Status flips draft→presented on PDF export. |
| #35 | Bugs 6+12+13 | Outlier symmetry refactor + singleton-outlier pass + time-of-sale appreciation (initial 5.0% default). Cutter Court correctly excluded as +111% singleton outlier on Candlestick. |
| #36 | Bug 14 | Lower default appreciationRatePerYear 5.0% → 3.0%. Fixed narrative hallucination root cause (LLM now sees per-comp line breakdown via computeAdjustmentsBatch + ADJUSTMENT ATTRIBUTION rules in system prompts). |
| (hotfix) | Model ID | Update deprecated claude-sonnet-4 snapshot to current. Packet generation routes back to clean. |

### Earlier today (sessions 2026-05-08 + 2026-05-11 morning)

All 11 original engine bugs shipped across PRs #25-#33. Bug catalogue:
- #25 Bug 5: PMV bounds invariant (Low ≤ anchor ≤ High; fallback anchor*0.93/1.08)
- #26 Bug 6 original: Active comp weight drops to 0.03 when only 1 Sold
- #27 Bug 4: Anchor sanity check
- #28 Bug 7: Subject self-listing exclusion
- #29 Bugs 8+9: Distance-tiered cascade + per-comp distance/direction
- #30 Bug 10: Cross-state deprioritization
- #31 Bugs 1+3+11: Per-feature parity scoring + outlier counter-cluster + strategy band cap
- #32 hotfix: Removed below_grade_unfinished_area from COMP_SELECT
- #33 Bug 2: GLA/basement separation per Fannie Mae URAR Form 1004

### Real-world verification status — ALL THREE SHIPPABLE

**Candlestick (6022 Candlestick Ln, Bent Creek SC, luxury 6BR)**: PMV $1,020,505 LOW confidence. Wyngate-anchored cluster math from 5 remaining comps after Cutter Court excluded as +111% singleton outlier. Aspirational $1,072K / PMV $1,021K / Event $969K all inside band.

**Cressingham (5022 Cressingham, Indian Land SC, 5BR ranch)**: PMV $670,983 MODERATE confidence. Five same-street Solds. Aspirational $700K / PMV $671K / Event $652K.

**Tyndale (215 Tyndale Ct, Somerset 28277)**: PMV $788K (tight confidence inferred from 4% spread). Aspirational $818K / PMV $788K / Event $767K.

Three properties across three different price tiers, three different market temperatures, three different data densities. All three landed with defensible math, honest confidence labels, narratives that don't lie about adjustments.

### Key engineering lessons captured today

1. **Dual-Supabase architecture confusion**: cma_reports lives in Core (`ioypqogunwsoucgsnmla`), MLS data in Listing Reports (`wdheejgmrqzqxvgjvfee`). Saved Code memory: "diagnose, don't create."
2. **Verify Postgres columns before specifying**. PR #32 hotfix lesson — column check via Supabase MCP before specifying any new field.
3. **Status enum is `draft | presented | archived`**. 'Presented' is canonical post-packet value.
4. **Carolina MLS field structure** (verified):
   - `above_grade_finished_area` = GLA
   - `below_grade_finished_area` = finished basement
   - `living_area = above_grade + below_grade` (SUM, includes basement)
   - **`below_grade_unfinished_area` does NOT exist** — comp-side unfinished basement is unrecoverable.
5. **Wyngate ground truth**: 7026 Wyngate Place, 29720. above_grade=3,266, below_grade_finished=1,689, total=4,955. Year built 2020.
6. **Appreciation defaults must reflect current market era**: 3.0% default for Charlotte metro 2024-2025. Hot submarkets (Fort Mill, Waxhaw) 4-5%. Cooling (Lancaster, eastern Union) 0-2%. Tunable per-deal in Rates UI.
7. **LLM narrative hallucination root cause** (Bug 14): LLM shown only adjustment totals + prose rollup will free-associate descriptions to fit dollar magnitudes. Fix: per-comp line breakdown + explicit ATTRIBUTION rules in system prompt.
8. **Model IDs in the Anthropic SDK go stale**: hardcoded versioned snapshots get deprecated. Code that calls the API needs to track current model id or use the latest alias. Standing rule: when adding LLM features, prefer the latest-version alias over snapshot dates.

### Test report IDs (regression baselines)

- `b4278470-e018-466e-b5a5-71d71cd06b2b` — fresh 6022 Candlestick (created 2026-05-11)
- `3a84f12c-dc91-407b-ac93-5efaa5c4d564` — Candlestick draft used through 2026-05-08
- `8023175d-37c8-45d7-95da-b9245abe4761` — original Candlestick
- 5022 Cressingham, 215 Tyndale Ct, 504 Redwine St, 5601 Medlin Rd as labeled

### Pickup checklist for next session

1. **STOP CODING on CMA tool for 24-48 hours**. Engine is appraisal-grade. Use it on a real listing this week.
2. **Mobile responsive pass (Enhancement 2)** — fix header overflow + page wrapper gutter on iOS Safari widths. 15-20 min CSS work. Not blocking desktop workflow but worth before agents start using it on phones.
3. **Other repos for CLAUDE.md ship-it specs** (priority order, deferred until CMA gets real-world reps):
   - `hgpg-transaction-manager`
   - `brain-app`
   - `charlotte-new-construction-nextjs`
   - `south-charlotte-report`
   - `homegrown-property-group-site`
4. **Mac mini Claude Code MCP setup** when Brian works there next.

### Deferred enhancement queue

- **Enhancement 2 — Mobile responsive pass** (NEW, surfaced 2026-05-11): header nav overflow, page wrapper gutter missing on /appraiser/new and probably elsewhere
- **Build-year premium**: dropped intentionally. Charlotte metro doesn't reliably price 2019 vs 2020 differently
- **Time-on-market adjustment for actives**: lower priority
- **Submarket-specific appreciation rates**: workflow item, agents tune per-deal in Rates UI

### Mac mini Claude Code MCP setup

Still needs MCP+GitHub PAT setup as iMac. When Brian works there next:

    claude mcp add --transport http --scope user --header 'Authorization: Bearer NEW_PAT_HERE' github https://api.githubcopilot.com/mcp/

Generate separate PAT named "Claude Code MCP - Mac mini". Alias `claude="claude --dangerously-skip-permissions"` already in iMac `~/.zshrc`. Mac mini needs alias added.

### Web Claude Code on iOS — viable workflow path

Discovered today: Claude Code on the web works from the iOS Claude app. Fire-and-forget pattern for bounded tasks. Terminal Claude Code remains better for iterative/verification work.

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

### Infra changes that affect other apps
- Resend custom SMTP wired into `HGPG Core` Supabase (project `ioypqogunwsoucgsnmla`)
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - API key stored under "Supabase HGPG Core" in Resend
  - Rate limit went from 2/hr to 30/hr (Resend default), can be raised
- Supabase project renames for hygiene:
  - `ioypqogunwsoucgsnmla` → "HGPG Core"
  - `ngdrliyjtqcwhhfrbxao` → "HGPG FUB Integration"
  - `wdheejgmrqzqxvgjvfee` → "HGPG Listing Reports + MLS"
  - `fkxgdqfnowskflgbuxhm` → "HGPG Signature + Relocation"

### Pickup notes
- Brain-app is live and working
- Resend API key is in 1Password ("Supabase HGPG Core SMTP")
- Brain-app local dev: `cd ~/brain-app && npm run dev`
