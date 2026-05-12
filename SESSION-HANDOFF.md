<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — New Construction phone capture spec locked, Claude Code prompt prepared

### What got done
- Confirmed: new construction site live at `newconstruction.homegrownpropertygroup.com` currently captures email + name on 4 lead forms; phone is NOT captured
- Verified via live curl - zero `type="tel"`, `name="phone"`, or phone copy markers in homepage HTML
- Locked 3 strategy decisions:
  - Tiered: optional on Guide/Quiz/Calculator, required on Builder Intro
  - Benefit-led copy, tailored per form (text guide link, text matches, text breakdown, 24hr callback)
  - FUB destination = standard `mobile` phone on contact, no custom field
- SMS speed-to-lead automation explicitly deferred - phone capture ships first, verify in FUB with a real lead, then SMS PR as separate session
- Full spec saved to brain at `projects/new-construction-phone-capture.md` (commit `e8a2e64`)
- CONTEXT.md updated to index the new project (commit `b9f5c91`)
- Claude Code prompt fully written, ready to paste

### Pickup notes for next session

**If phone capture has shipped:** Verify in FUB that a real Builder Intro lead has phone attached as mobile, then prep the SMS speed-to-lead PR (FUB Automations 2.0 rule, 90-second SMS auto-response on Builder Intro source). Update `projects/new-construction-phone-capture.md` status to SHIPPED, add follow-up notes.

**If phone capture has NOT shipped:** The Claude Code prompt is the deliverable. It's verbatim in the previous Claude conversation (2026-05-12) and the full spec lives at `projects/new-construction-phone-capture.md`. Pickup is one step: paste the prompt into Claude Code on Mac, let it run, review PR, merge.

**Smoke test once shipped (before merging PR):**
1. Guide Delivery, no phone → 200, FUB email-only
2. Quiz, phone `(704) 555-1234` → FUB has `+17045551234` as mobile
3. Calculator, phone `12345` → inline error, blocked
4. Builder Intro, empty phone → inline error, blocked
5. Builder Intro, valid phone → FUB + Meta Test Events both fire
6. Meta Events Manager → Test Events → confirm `ph` parameter on Lead payloads

### Repo + infra reference
- Repo: `HGPG1/charlotte-new-construction-nextjs` (branch `main`)
- Vercel auto-deploys on merge
- Meta Pixel ID 1880396459290092 already wired, 6 funnel events live
- Shared FUB helper: `src/lib/fub.ts`
- FUB API key already in Vercel env

### Open questions, none right now
None. Spec is locked, ready to execute.

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

(preserved below for reference)

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
