<!-- Last Updated: 2026-05-13 -->

# Session Handoff

## Last session: 2026-05-13 — SMS speed-to-lead SHIPPED 🟢 (live test pending)

### What got shipped today

**Stage 1 code (PR #2 merged):**
- `customSmsConsent: 'YES' | 'NO'` field now written to FUB on every Builder Intro / Guide / Quiz / Calculator submission
- Commit `5fe1ce9` on main, Vercel auto-deployed to prod
- Verified live - test contact `SmsTest Field` has `Sms Consent: YES` in FUB

**Stage 2 FUB config:**
- Custom field `Sms Consent` created in FUB Admin (dropdown YES/NO, API name `customSmsConsent`)
- Automation `New Construction - Builder Intro - Post SMS Workflow` built (Manual trigger, 5-min wait, Create Task) - ENABLED
- Lead Flow rule built on source `Website - New Construction`: Tags include `Builder:`, distributes to Brian, attaches Automation, sends initial text instantly

### Architecture changed from original spec - read project file

Original spec assumed FUB Lead Flow + Action Plan. WRONG. The shipped reality:
- FUB Lead Flow conditions cannot filter on custom fields (only Tags, Price, City, State, ZIP, MLS, Phone)
- FUB Automations 2.0 cannot send SMS (no Send Text step type)
- ONLY native auto-SMS path = Lead Flow's "initial text message" field (NOT Action Plans, NOT Automations)
- Action Plans are deprecated per HGPG standing rule, so the Lead Flow native SMS path is the only viable option

The TCPA defensibility model now lives at the FORM layer (double-gated client + server consent enforcement), with `customSmsConsent` + event-body line as audit artifacts. Lead Flow trusts upstream enforcement. Full details in `projects/new-construction-sms-speed-to-lead.md`.

### Open item: live end-to-end test (5 min when back at desk)

Brian hadn't run a live test yet. Procedure:
1. Submit Builder Intro from incognito with REAL phone (not 555-pattern - those are flagged invalid in FUB)
2. SMS should arrive from `(980) 261-9222` within seconds
3. Task should appear in FUB at the 5-min mark
4. Reply STOP - should opt-out cleanly

Brain file has full troubleshooting guide if anything fails.

### Key learnings captured in project file

- FUB Lead Flow condition limits (Tags / Price / location / MLS / Phone only)
- FUB Automations 2.0 has no Send Text step type
- FUB custom field API names preserve uppercase runs in labels (we got bit once - rename label to control casing)
- FUB tag matching is case-insensitive on partial-match
- FUB send-from number for new construction: (980) 261-9222
- 555-pattern phones get flagged invalid

### Pickup notes for next session

If Brian comes back and says "the test worked": flip the project status from 'pending live test' to fully SHIPPED in the project file, close out.

If the test failed: troubleshooting steps in the project file under 'End-to-end test plan' section. Most likely failure modes: Automation not enabled, SMS copy formatting issue, Lead Flow rule positioned below Default Rule.

If Brian wants to extend the Automation: parked follow-up to add a Day-1 email step after the task. Currently it's just SMS + 5-min task.

If Brian wants the FUB label to read 'SMS Consent' (caps) instead of 'Sms Consent': 1-line code patch to change the customSmsConsent key to customSMSConsent. Purely cosmetic - leave for now.

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
