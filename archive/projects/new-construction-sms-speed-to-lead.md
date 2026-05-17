<!-- Last Updated: 2026-05-14 -->

# New Construction Site - SMS Speed-to-Lead (Builder Intro)

**Status:** 🟢 SHIPPED 2026-05-13. Live end-to-end test 📋 PARKED for 2026-05-19+ (paired with phone capture rate review)
**Repo:** `HGPG1/charlotte-new-construction-nextjs` (branch `main`)
**Domain:** `newconstruction.homegrownpropertygroup.com`
**Driver:** Builder Intro is the hottest funnel surface (hand-raise). Instant SMS auto-response on submission so HGPG is first in the door before the builder rep calls.

## Final architecture (different from original spec - read this)

Original spec called for FUB Lead Flow + Action Plan. That was wrong. The actual shipped architecture:

```
Builder Intro lead lands at /api/builder-lead route
    -> FUB contact created with customSmsConsent='YES', tag 'Builder: X', source 'Website - New Construction'
    -> Lead Flow rule matches (Tags include 'Builder:')
    -> Native initial text message fires immediately (NOT via Action Plan)
    -> Automation 'New Construction - Builder Intro - Post SMS Workflow' kicks off
        -> Wait 5 minutes
        -> Create Task 'Call Builder Intro lead', assigned to agent on contact
```

**Why this architecture differs from the spec:**
- FUB Lead Flow condition dropdown only exposes: Tags, Price, City, State, ZIP, MLS, Phone Number. **Custom fields are NOT available as Lead Flow conditions.** Verified live 2026-05-13.
- FUB Automations 2.0 do NOT have a native "Send Text" step type. Only Lead Flow's "initial text message" can send native SMS.
- Action Plans are deprecated per HGPG standing rule. So Lead Flow's native SMS field is the only on-architecture path.
- The Automation handles post-SMS workflow (wait + task). Trigger: Manual (called by Lead Flow).

## TCPA defensibility model (important - read before changing anything)

**Consent gate is enforced at the FORM, not at Lead Flow.** This is intentional and defensible. Four layers of audit trail:

1. **Form layer:** Builder Intro form requires phone (required field) and SMS consent (required checkbox). Submit is double-gated client-side + server-side. Server returns 400 if either missing.
2. **API layer:** Server route only writes to FUB if both checks pass.
3. **FUB contact:** `customSmsConsent` field = `YES` on every Builder Intro contact reaching FUB (since lower layers reject anything else).
4. **Event body:** `SMS Consent: YES` line written into the FUB event message body as a second audit artifact.

Lead Flow conditions on `Tags include "Builder:"` only. It does NOT re-check consent at the Lead Flow layer because:
- It can't (FUB Lead Flow can't filter custom fields)
- It doesn't need to (every lead reaching this rule has already cleared the form gate)

**CRITICAL: If Builder Intro is ever changed to make phone or consent optional, this Lead Flow rule MUST be updated** to add a re-check condition. Possible paths if that day comes:
- Add a tag like `SMS Consented` written from code when consent=true, condition Lead Flow on that tag
- Move SMS send out of Lead Flow into the API route (LoopMessage from Builder Intro route)
- Restrict Lead Flow rule to a more specific tag set

## What's live in FUB (built 2026-05-13)

### Custom field
- **Label:** Sms Consent (lowercase 'ms' because FUB regenerates API name from label - this matters)
- **API name:** `customSmsConsent`
- **Type:** Dropdown
- **Choices:** YES, NO
- Verified via `GET /customFields` API

### Automation
- **Name:** `New Construction - Builder Intro - Post SMS Workflow`
- **Trigger:** Manual
- **Step 1:** Wait 5 minutes
- **Step 2:** Create Task `Call Builder Intro lead`, type=Call, assigned to "Agent assigned to the contact"
- **Status:** Enabled

### Lead Flow rule
- **Location:** Source group `Website - New Construction` -> Rule 1
- **Condition:** `Tags include "Builder:"` (partial-match style filter - catches `Builder: DR Horton`, `Builder: Lennar`, all future builders)
- **Distribution:** Brian McCarron
- **Automation:** `New Construction - Builder Intro - Post SMS Workflow`
- **Initial text message:**
  ```
  Hi %contact_first_name%, Brian from Home Grown Property Group. Thanks for the builder intro request. I'll reach out shortly with next steps. Reply STOP to opt out.
  ```
- **Delay:** 0 minutes (instant)
- **Send-from number:** `(980) 261-9222` (FUB-assigned)
- **Positioned above Default Rule**

## Stage 1 code (shipped earlier today via PR #2, commit 5fe1ce9)

`src/lib/fub.ts` writes `customSmsConsent: 'YES' | 'NO'` to the person payload on the FUB Events API call. Always set (never undefined / null). Existing 'SMS Consent: YES/NO' line preserved in the event message body as second artifact. Consent label copy already exceeded TCPA bar (sender, scope, rates, STOP, frequency, privacy link) - no copy changes needed.

## End-to-end test plan

Pending one live test (Brian had to jet). When run:
1. Submit Builder Intro from incognito with REAL phone number (NOT 555-pattern - those are flagged invalid in FUB and SMS won't send). Use first name `LiveTest`, email pattern `live-test-<timestamp>@homegrownpropertygroup.com`.
2. Phone should receive SMS from `(980) 261-9222` within seconds: `Hi LiveTest, Brian from Home Grown Property Group. Thanks for the builder intro request...`
3. Open contact in FUB - confirm: phone shown, `Sms Consent: YES`, tag `Builder: <picked builder>`, event in timeline showing the SMS that went out.
4. Wait 5 minutes - confirm task `Call Builder Intro lead` appears, assigned to Brian.
5. Reply STOP to the SMS - confirm FUB marks contact opted-out (native FUB behavior).

If anything fails:
- No SMS arrived? Check Vercel runtime logs for the `/api/builder-lead` route, check FUB contact for the tag and consent field, check Lead Flow rule order (must be above Default).
- SMS arrived but no task at 5 min? Automation didn't fire - check Automation is Enabled, check Automation dropdown on Lead Flow rule is set to the right Automation.

## Key learnings (FUB internals worth remembering)

1. **FUB Lead Flow conditions are restricted to: Tags, Price, City, State, ZIP, MLS, Phone Number.** Custom fields, source, event type are NOT filterable at Lead Flow. Don't spec gates that depend on them.
2. **FUB Automations 2.0 don't have a native Send Text step.** SMS only happens via Lead Flow's "initial text message" feature, or via Action Plans (which are deprecated).
3. **FUB custom field API names preserve uppercase runs in the label.** Label `SMS Consent` -> API name `customSMSConsent` (NOT `customSmsConsent`). Label `Sms Consent` -> API name `customSmsConsent`. **Always run `GET /customFields` and verify the API name after creating any new field.** This bit us once already.
4. **FUB tag matching is case-insensitive in the partial-match style** (`Builder:` matches `Builder: DR Horton`). Good for forward-compatible filters.
5. **FUB Lead Flow doesn't auto-recheck custom fields.** Consent gates have to live at the application layer if you need re-validation.
6. **FUB send-from number for new construction:** `(980) 261-9222`. Replies route into FUB.
7. **555-pattern phones get flagged invalid in FUB and SMS won't actually send.** Use real phones for end-to-end tests.

## Verification commands

```
curl -s -u "fka_0cyqBULqHHKqrZiEJH6Njl7jqDkBBNTUYJ:" https://api.followupboss.com/v1/customFields | python3 -m json.tool | grep -A2 -i "sms"
```

```
curl -s -u "fka_0cyqBULqHHKqrZiEJH6Njl7jqDkBBNTUYJ:" "https://api.followupboss.com/v1/people?q=EMAIL_TO_SEARCH" | python3 -m json.tool
```

## Follow-ups (parked)

- Live end-to-end test (5 min, real phone needed) — 📋 PARKED for 2026-05-19+ window, pair with `new-construction-phone-capture-rate-review.md`
- Decide whether to drop the `Sms Consent` label back to `SMS Consent` (caps) and patch the code in a 1-line follow-up PR. Today the FUB UI label reads lowercase-ms. Purely cosmetic. Operational over polish for now.
- Consider extending the Automation: add a Day-1 follow-up email step after the task. Currently just SMS + task.

## References

- Phone capture parent project: `projects/new-construction-phone-capture.md` (shipped 2026-05-12)
- Stage 1 PR (merged 2026-05-13): https://github.com/HGPG1/charlotte-new-construction-nextjs/pull/2, commit 5fe1ce9 on main
- FUB API key (in env, also in this file for terminal verification): `fka_0cyqBULqHHKqrZiEJH6Njl7jqDkBBNTUYJ:`
