<!-- Last Updated: 2026-05-12 -->

# Session Handoff

## Last session: 2026-05-12 — Phone capture SHIPPED 🟢 on newconstruction.homegrownpropertygroup.com

### What got shipped
- PR #1 on `HGPG1/charlotte-new-construction-nextjs` merged to main
- Merge commit: `8bca9ac` (verified)
- Prod deploy: `dpl_2UNJu3ByZa5tbdG3WK9NDAfqdnZx` READY
- Tiered phone capture: optional on Guide/Quiz/Calculator, required on Builder Intro
- E.164 normalization in `src/lib/fub.ts` with junk-input null-out
- Per-form benefit-led helper copy
- Meta CAPI `ph` hashing was already correct - just got Last Updated comment

### Important reframe
Phone capture was NOT a greenfield build. Phone was already captured and required across all 4 forms before this PR. This was a **tiered downgrade** with proper normalization + per-form copy. Brain originally framed this wrong - corrected in `projects/new-construction-phone-capture.md`.

### Key implementation notes (read these before touching new construction lead capture again)
- Only 2 form components (LeadCaptureModal shared by 3 forms, BuilderLeadModal standalone)
- SMS consent checkbox is now CONDITIONAL on phone presence (hidden until user types a digit)
- `normalizePhone()` in `src/lib/fub.ts` is canonical; `normalizePhoneClient` duplicated in client components because lib/fub.ts imports next/server
- FUB email now uses `type: 'home'`; phone uses `type: 'mobile'`
- New "Phone: not provided" line in FUB note text distinguishes email-only leads from phone-having-but-declined-SMS leads

### Preview review (before merge)
Brian ran fast-path 3-check, 2 passed (conditional SMS consent, Builder Intro block), 3rd (FUB record clarity) deferred to first real lead.

### Pickup notes for next session

**Primary follow-up: SMS speed-to-lead automation PR.** Parked decision before building:
- Gate SMS auto-response on the SMS consent checkbox (legal-clean / defensible)
- OR fire on Builder Intro submission regardless (faster, riskier on TCPA)
- Builder Intro requires both phone + consent today, so functionally equivalent now
- Recommendation: gate on consent for audit trail defensibility

**Secondary check (low priority):** Next time Brian is in FUB, eyeball one phone-having and one email-only lead from new construction to confirm the "Phone: not provided" vs `(Mobile)` display reads cleanly. If it doesn't, 5-min string change.

### Live state
- New construction site fully live with shipped phone capture
- Meta Pixel + CAPI firing all 6 funnel events + `ph` param on phone-submitting Lead events
- Vercel auto-deploy from main working as expected
- FUB lead capture working; new contacts will land with proper mobile phone field when provided

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
