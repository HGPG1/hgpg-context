<!-- Last Updated: 2026-05-13 -->

# Session Handoff

## Last session: 2026-05-13 (full day) - Three NC site builds shipped 🟢

Today was a heavy build day on `newconstruction.homegrownpropertygroup.com`. Three production deploys, four brain commits.

### What shipped today (in order)

**1. FUB customSmsConsent field (PR #2 merged earlier today)**
- `customSmsConsent: 'YES' | 'NO'` written to FUB on every lead form submit
- Unblocks Lead Flow conditioning on consent
- Consent copy audit confirmed existing checkbox already exceeds 4-element TCPA bar

**2. SMS speed-to-lead (FUB UI config, no code)**
- Built `Sms Consent` custom field in FUB (dropdown YES/NO, API name `customSmsConsent`)
- Built Automation `New Construction - Builder Intro - Post SMS Workflow` (Manual trigger, 5-min wait, Create Task)
- Built Lead Flow rule on source `Website - New Construction`: Tags include `Builder:`, attaches Automation, sends initial text instantly
- See `projects/new-construction-sms-speed-to-lead.md` for the full architecture (it changed mid-session from the original spec because FUB Lead Flow can't filter custom fields)
- **Outstanding:** live end-to-end test from Brian's real phone

**3. Scout admin cleanup (PR #3 merged)**
- Added `scout_status` column to `nc_builders` (crawl | email_only | paused)
- Scout admin tab now filters where `scout_status = 'crawl'`: 23 -> 19 builders shown
- KB Home incentives_url fixed (404 -> working Indian Land page)
- 4 builders flipped to email_only (Eastwood, Mattamy, Stanley Martin, Taylor Morrison) - they're blocked by 403 or JS-required, ingested via Apps Script email pipeline instead
- Status badge + edit field on the Builders tab so Brian can flip scout_status from the UI without SQL
- See `projects/new-construction-scout-admin-cleanup.md`

### Open / pending

**Live end-to-end test for SMS speed-to-lead.** Brian was at the office and didn't run it. Procedure:
1. Submit Builder Intro from incognito with REAL phone (not 555-pattern - flagged invalid in FUB)
2. SMS should arrive from `(980) 261-9222` within seconds
3. Task should appear in FUB at the 5-min mark
4. Reply STOP to verify opt-out

If anything fails, troubleshooting guide is in `projects/new-construction-sms-speed-to-lead.md`.

### Parked follow-ups (low priority)

- **Phone capture rate review** (run 2026-05-19 to 2026-05-26) - see `projects/new-construction-phone-capture-rate-review.md`
- **"For Builders" footer link** to /builder-submit - bundle with next NC site touch, see `projects/new-construction-builder-submit-footer-link.md`

### Key learnings captured in brain

- FUB Lead Flow can only filter on Tags, Price, City, State, ZIP, MLS, Phone (NOT custom fields, NOT source, NOT event type)
- FUB Automations 2.0 have no native Send Text step - SMS only happens via Lead Flow's "initial text message" field
- FUB custom field API names preserve uppercase runs in labels (`SMS Consent` -> `customSMSConsent`, `Sms Consent` -> `customSmsConsent`). Always GET /customFields to verify after creating
- FUB send-from number for new construction: (980) 261-9222
- 555-pattern phones are flagged invalid in FUB and won't actually receive SMS
- NC Scout builders fall into 3 ingestion buckets (crawl / email_only / paused). Filter at API layer, not just UI
- Builder-rep submission form lives at `/builder-submit` (unlinked from nav by design)

### Pickup notes for next session

When Brian returns: ask if the SMS live test happened. If yes and it worked, flip SMS project status from "pending live test" to fully shipped. If it didn't work, troubleshooting guide in the project file.

If Brian asks about adding more builders to the NC site: he ALREADY has 23, which is dense coverage. The original "find missing builders" question was tangential to his real ask (hide non-crawlable ones), and the actual cleanup shipped today. Don't re-litigate adding builders unless he explicitly asks.

---

## Prior session: 2026-05-12 — Phone capture SHIPPED 🟢

PR #1 merged, prod deploy live. Tiered phone capture (optional on Guide/Quiz/Calculator, required on Builder Intro). Was reframed mid-session - phone was already captured everywhere, the real work was tiering. Two of three preview-deploy checks passed, third (FUB record clarity) deferred to first real lead. Full details in `projects/new-construction-phone-capture.md`.

---

## Prior session: 2026-05-06 — Brain App MVP shipped 🟢

(preserved for reference)

- New Vercel project: `brain-app` on team `team_FietQPKCmnyioG2n0FdteQCV`
- New repo: `HGPG1/brain-app` (private)
- Live at: `https://brain.homegrownpropertygroup.com`
- Stack: Next.js 16.2.4, Tailwind v4, CodeMirror 6, Supabase Auth (magic link)
- Single-user lock: `BRIAN_EMAIL=brian@homegrownpropertygroup.com` allow-list
- Resend custom SMTP wired into `HGPG Core` Supabase, affects ALL apps using that project
- Supabase project renames for hygiene (see prior handoff)
