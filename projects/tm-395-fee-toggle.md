<!-- Last Updated: 2026-05-08 (originally parked 2026-05-06) -->

# TM: $395 Transaction Fee Toggle

- **Status:** 🔵 Build spec parked, awaiting greenlight
- **Origin:** Don feedback batch resolved 2026-05-06 (commit `b9fa0deb`)
- **Repo:** HGPG1/hgpg-transaction-manager
- **Live URL:** closings.homegrownpropertygroup.com

## Why this is parked

Don's feedback for the $395 transaction fee shipped as a copy/template fix in the last batch (commit `b9fa0deb`). The actual buyer/seller-facing toggle that lets HGPG enable or disable the fee per deal was scoped but not built. The wizard fix Don asked for was the surface-level need; the toggle is the structural improvement that makes the whole feature feel intentional.

## Build spec

### Database

Single boolean column on `transactions`:

    ALTER TABLE public.transactions
      ADD COLUMN IF NOT EXISTS hgpg_fee_applies boolean DEFAULT true;

- Default `true` (matches current behavior — fee applies by default)
- Nullable: no — every deal has an answer
- Migration name: `add_hgpg_fee_applies_to_transactions_2026_xx_xx`

### Wizard surface

Add a single checkbox to the buyer + seller intake wizards:

- Label: `[ ] $395 HGPG transaction fee applies to this deal`
- Default: checked
- Helper text: small grey, "Uncheck if waiving for VIP / referral / past client / etc."
- Position: in the financials section, near commission/fee fields

When unchecked, the value persists to `transactions.hgpg_fee_applies = false`.

### Task auto-generation

Currently the verify-fee task is auto-inserted into every deal. After this build:

- If `hgpg_fee_applies = true`: insert "Verify $395 Fee" task as today
- If `hgpg_fee_applies = false`: do NOT insert the task

Task template name: query `task_templates WHERE template_name LIKE '%Verify $395 Fee%'` to retrieve at insert time. The template itself is unchanged — only the conditional gate is new.

### Edit surface

Deal page should expose the toggle in the edit panel so Don/Brian/Ashley can flip it after the wizard if a fee waiver is decided later. When flipped from `true` to `false` on an existing deal:

- The "Verify $395 Fee" task: should it be deleted, marked complete, or left as-is?
- Recommend: leave as-is. If the task already exists, deleting it loses the audit trail. If it's not yet started, Don can manually mark it complete with a note.

When flipped from `false` to `true`: insert the task fresh.

### UI display

The financials summary on the deal page should show:

- If applies: "$395 HGPG fee" line item
- If not: hide the line entirely (don't show "$0" or "waived" — just absent)

### Notification copy

Buyer/seller wizard receipt emails should reflect the toggle:

- If applies: include the fee line in the financial summary as today
- If not: omit it cleanly

Same for milestone notifications that reference total funds — recalculate without the fee.

## Open question for Don before building

Cosmetic, not blocking: does the checkbox label want to mention buyer agency vs. listing agreement specifically, or stay generic? Current spec uses generic "$395 HGPG transaction fee."

## Constraints

- Buyer-side and listing-side wizards both get the toggle — fee applies in both directions
- The toggle is per-deal, not per-team-member or per-client
- Existing deals already have the verify-fee task; this build does NOT retroactively touch them. Don can manually flip + delete on a per-deal basis if needed.

## Estimated effort

- Database migration: 5 min
- Wizard checkbox + persistence: 30 min (both wizards)
- Conditional task insert logic: 20 min
- Deal page edit surface: 30 min
- Receipt email + milestone notification copy: 30 min
- QA on a test deal: 20 min

Total: ~2.5 hours of focused build, single batch, single commit.

## Pickup notes

- The Don feedback that triggered this spec was about copy on the receipt email. That copy fix already shipped. This spec is the structural follow-up.
- When this build runs, the migration name should slot into the existing TM migration history without conflicts. Check most-recent migration name first.
- Test deal `15d83bd2-3c22-44d2-a427-18368713beb3` (4421 Magnolia Point Drive) already has `hgpg_fee_applies` semantically true (default state). Any unit testing of "fee not applying" needs a fresh test deal or a manual flip.
