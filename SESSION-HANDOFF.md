<!-- Last Updated: 2026-05-11 -->

# Session Handoff

## Next session queued: Buyers Guide instrumentation

**Goal:** ship Meta Pixel + CAPI, NeverBounce email validation, and Manus 301 redirect on the Buyers Guide. Sellers Guide already has all three; Buyers Guide does not.

**Verified state (2026-05-11):** production bundle at `https://buyersguide.homegrownpropertygroup.com/assets/index-*.js` has zero references to `fbq`, `connect.facebook.net`, `neverbounce`, or any validate-email/CAPI endpoint. All three items are genuinely pending.

**Session size estimate:** 2-3 hours single sitting if playbook + assets are pre-staged.

### Prereqs to stage BEFORE kicking off Claude Code

1. **Pixel ID for Buyers Guide** — playbook says one Pixel per site. Either create a new Pixel in Meta Business Suite for the Buyers Guide, OR reuse an existing unassigned one. Sellers Guide and New Construction each have their own; Buyers Guide needs its own.
2. **NeverBounce API key decision** — Sellers Guide already has one in Vercel env. Either reuse the same key on Buyers Guide (single NeverBounce account, no new key needed, just add to Buyers Guide Vercel env), or provision a fresh key for per-site quota visibility. Reuse is simpler.
3. **Manus original URL** — currently flagged unknown in `projects/buyers-guide.md`. If Brian can produce it, 301 redirect is 5 min. If not, skip and leave pending — don't block the other two.

### Kickoff prompt (paste into Claude Code in the Buyers Guide repo)

Prompt is in `projects/buyers-guide.md` under "Pending instrumentation session" section — copy from there.

### Verification gates the agent must pass

- `npm run build` clean, `npx tsc --noEmit` clean
- After deploy: bundle grep for `fbq` > 0, `neverbounce` > 0, Pixel ID present as 15-16 digit string
- POST to `/api/validate-email` returns real NeverBounce JSON (not SPA index.html fallback — earlier check showed Vercel rewrites currently swallow unknown routes)
- POST to `/api/meta/capi` returns real CAPI response
- End-to-end test: NeverBounce validates → browser Pixel fires → CAPI fires server-side with matching `event_id` → FUB receives lead via Events API
- Meta Events Manager Test Events confirms browser + server events deduplicate (not double-counting)

---

## Last session: 2026-05-11 — Brain reconciliation + transaction-pdfs cleanup + project list pass

### What got done

Reconciled CONTEXT.md against project files (which were ahead), closed six items:

1. **Sherlock 403 on Transaction Manager** — was already marked closed in `projects/transaction-manager.md` on 2026-05-09. Fix was likely security/auth related; no specifics logged. If recurs, start with API key scope.
2. **Mac Mini GitHub auth / Listing Report Portal** — already 🟢 in project file (2026-05-06 favicon push verified).
3. **Exposed GitHub PAT rotation** — closed. Fine-grained brain-app PAT (scoped to `hgpg-context`, contents:write only) superseded the broad old token.
4. **CMA Engine MLS Grid auto-pull** — already marked mature in `projects/cma-engine.md` on 2026-05-09.
5. **NC office routing in ReZEN builder** — **verified live**. Code at `app/api/rezen/create-transaction/route.ts:160` uses `txn.state?.toUpperCase() === "NC" ? REZEN_OFFICE_ID_NC : REZEN_OFFICE_ID`. Both env vars confirmed on Vercel Production.
6. **transaction-pdfs bucket flip → private** — fully closed. Investigation revealed bucket was flipped via MCP on 2026-05-09 (`schema_migrations` version `20260509173839`), code on main since PR #7 (`e45e3c8`), but the `.sql` migration file was never committed alongside the code. Today: re-applied idempotent migration via MCP (`20260511210146`), staged file, Brian committed and pushed from Mac (`2eb9794`). Bucket, DB ledger, repo file, and code all aligned.

### Brain hygiene fixes

- Rebuilt `projects/brain-app.md` (had been clobbered with handoff-style content earlier — restored as proper project spec with full Write API documentation)
- Updated CONTEXT.md "active right now" to include FUB AI Agent, Sellers Guide, Buyers Guide, Brain App (had been missing or buried)
- **Buyers Guide false-close averted:** during recap, Brian believed Pixel + CAPI + NeverBounce were done. Live-site grep against `buyersguide.homegrownpropertygroup.com` proved otherwise. Items remain open. Saved drift in the wrong direction.

### Open thread
- `~/Downloads/fix-admin-page.sh` may have unpushed UI changes for Listing Report Portal (SocialPostsManager dark navy headings, MagicLinkCard regenerate placement, admin reorder, zero-value stat hiding). Check next time portal is touched.

### Pickup notes for next session (other than Buyers Guide above)

Real build items remaining:

- **CMA Engine** — awaiting Taylor stress test before declaring production-ready
- **$395 fee toggle** — structural build still parked (commit b9fa0deb)
- **Transaction Manager** — ongoing Don feedback batches
- **FUB AI Agent** — flip strategy decision pending (`agent_enabled=true` vs sustained manual-approve), plus 4,340 unscored eligible leads sweep
- **Sellers Guide** — FUB Automation 2.0 on `sellers-guide-2026` tag not yet built (blocks ad-spend scaling)

Non-build:
- .net Google Workspace migration to .com (do not proactively remind)

### Brain hygiene notes

Three lessons worth keeping:

1. **CONTEXT.md drifts** when project files get updated independently. Worth running a reconciliation pass every couple weeks. Quick check: `diff` what CONTEXT.md flags as blockers/active vs. what the actual project files say.
2. **Migrations applied via MCP need their .sql files committed too.** The DB ledger (`schema_migrations`) and the repo (`supabase/migrations/`) are two separate sources of truth. `apply_migration` only updates the former. When using MCP to ship DB changes, always commit the equivalent `.sql` file to the repo in the same workflow, otherwise a future `supabase db reset` will lose the change.
3. **Don't trust assertions about "this is done" without verifying against the live system.** Today: Brian thought Buyers Guide instrumentation was done; live bundle grep proved otherwise. Cost of verification is near-zero; cost of closing real work as done is real.

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
- Resend custom SMTP wired into `HGPG Core` Supabase
  - Sender: `noreply@homegrownpropertygroup.com`, name: HGPG
  - Rate limit 2/hr → 30/hr
  - Affects ALL apps using this Supabase: TM, CMA, TC Concierge, brain-app
- Supabase project renames for hygiene
- Supabase `HGPG Core` redirect URLs added for `brain.homegrownpropertygroup.com/**` and `http://localhost:3000/**`
