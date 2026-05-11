<!-- Last Updated: 2026-05-11 -->

# SESSION-HANDOFF

## Where we are

**Sessions 7 + 8 + 9 (2026-05-11) shipped.** FUB AI Agent is feature-complete for v1 buyer-only Brian-only pilot:
- Multi-field paragraph stitching architecture live (session 7)
- All 3 seller templates with too-long paragraphs fixed (session 8)
- Buyer template diversity via .vN variants + random pick (session 9)

agent_enabled=false in safe state. Ready to flip true for sustained operation.

Status: 🟢 Operational, feature-complete v1.

## Today's commits

**TM (HGPG1/hgpg-transaction-manager) main:**
- 504f116 - multi-field paragraph stitching architecture
- c8f0d51 - constraint-violation bug fix (paragraphs=null on blockReason)
- 808c456 - selectTemplate matches base scenario + .vN variants

**Brain (HGPG1/hgpg-context) main:**
- 74f81c7 - session 7 SESSION-HANDOFF
- 7f10686 - session 7 project file
- 61572e1 - CONTEXT.md update
- bd600f3 - session 8 project file entry
- f18eed0 - session 8 SESSION-HANDOFF
- bd02910 - session 9 project file entry
- (this commit) - session 9 SESSION-HANDOFF

**Templates touched in Supabase (`ioypqogunwsoucgsnmla`):**
- Templates 2, 9, 11 — paragraph 3 trimmed under 250-char cap
- Templates 15, 16 — new buyer variants inserted

## Open queue (post-session-9)

agent_enabled=false. Nothing going out without manual approve.

| Draft | Lead | Template | Notes |
|---|---|---|---|
| 17 | Jesse Hernandez | seller:listing_thinking | approved + sent (Phase 7 verification, real outbound) |
| 23 | Jesse Hernandez | seller:listing_thinking | dupe with 17 - reject |
| 24 | John Miller | seller:seller_report_engaged | clean |
| 25 | Jay Miller | seller:seller_report_engaged | clean |
| 26 | Rachel Delmore | seller:listing_thinking | clean |
| 27 | Gerard Marmo | buyer:viewed_listing_recent.v3 | Brian in active manual convo - reject |
| 28 | Seanna Mackey | buyer:viewed_listing_recent.v3 | clean |
| 29 | Anthony Scott | buyer:viewed_listing_recent.v1 | clean |
| 30 | Tamela Karnazes | seller:seller_report_engaged | clean |
| 31 | Sam Russell | buyer:viewed_listing_recent.v3 | clean |
| 32 | Jason Smith | seller:listing_thinking | newly classified seller |
| 33 | Leigh Waldrep | seller:listing_thinking | newly classified seller |
| 34 | Chris Smith | buyer:viewed_listing_recent.v1 | newly classified buyer |
| 35 | Tracy Shrum | seller:seller_report_engaged | clean |

12 fresh pending drafts ready for queue review. Good test rig for Don when he starts using the queue UI.

## Pick up here (session 10 priorities)

Decision time. Three paths:

1. **Flip agent_enabled=true and start the manual-approve ramp.** All technical work is done. Daily cap is 10. Don could start reviewing the queue tomorrow morning. Optionally also flip auto_below_threshold=true to let high-confidence drafts auto-approve.

2. **Build the Don-facing dashboard first.** Currently the queue UI is the only interface. A summary view (drafts this week, approve rate, reject reasons, response signal) would help Don and Brian see whether the agent is working.

3. **Backfill scoring on 4,340 unscored eligible leads.** This 10x+ the candidate pool and is the biggest single lever for "more drafts" volume. Currently the pool feels small because most leads don't have a score yet.

## Critical context to carry forward

- agent_enabled blocks BOTH cron AND manual approve via outboundGate. Flip true for sends.
- FUB Automation 2.0 delivers email as plaintext (or mail clients pick plaintext). NEVER use `<br>` in templates - real newlines only.
- FUB UI Merge Fields dropdown produces `%snake_case%` tokens. Don't type them.
- Test rig: FUB person 27764 (Brian in Sphere, pond 9 excluded).
- gerardmarmo@yahoo.com (FUB 23552) - Brian in active manual conversation. Reject any agent drafts targeting him.
- Template paragraph sweep query lives in projects/fub-ai-agent.md "Session 8" entry.
- Variant naming convention: `v1.warm.scenario_name.channel` (base) + `.v2`/`.v3`/... for variants. Selector matches all .vN suffixes via regex and random-picks.

## Backlog (post-v1)

- More buyer/seller template variants (only viewed_listing_recent has variants; other scenarios still single-template)
- Hot tier templates (score >= 40 → distinct templates)
- iMessage seller variants
- Cooldown re-touch templates (second-attempt copy)
- 4,340 unscored eligible leads scoring sweep
- Don/Brian-facing dashboard
- FUB UI custom fields 157-162 visibility to admins-only
- Normalize hideIfEmpty across fields 157 vs 160/161/162 (cosmetic)
- Stale tag cleanup on person 27764 (`agent_draft_email_` trailing underscore)
- Drop body column after deprecation window
- v2 features: brokerage oversight expansion (Ashley/Brenda/Taylor + Don unified queue), tier-based behavior, smart channel pick, production inbound classifier
