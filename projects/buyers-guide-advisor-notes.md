# AdvisorNote content review

<!-- Last Updated: 2026-05-19 -->

**Source:** `AdvisorNote-C4olMM97.js` — lazy chunk loaded only when AdvisorMode is active. Pulled live 2026-05-19 from production at `buyersguide.homegrownpropertygroup.com`.

**What this file is for:** Every AdvisorNote that renders only in Advisor Mode (i.e. content visible to the agent during a buyer consult but never to the buyer themselves). Treat this file as the canonical inventory. Before any HGPG agent (Brian, Brenda, Ashley, Taylor) uses Advisor Mode in a live consult, review the points below and flag anything that needs a rewrite.

**How to flag a line:** add a `🔴 REVIEW:` prefix in this file with a comment about what to change. Next time we touch the Buyers Guide repo we'll batch the edits into a single deploy.

**Total:** 40 talking points across 12 sections, spanning 4 buyer-facing pages.

---

## /quiz

### Why we use the buyer quiz
- The five questions surface the real decision drivers in two minutes. Use the answers to set the meeting agenda.
- Ask the buyer to read each option aloud so you both hear their reasoning.
- If they hesitate between two options, that gap is the first thing to explore offline.
- The construction question often exposes whether the buyer has a builder bias from a friend or family member.

### Buyer type framing
- First-time buyers need the loan-product conversation before tours; relocators need a city orientation.
- Move-up families almost always have an equity question. Pivot to the Calculator equity module.
- Investors should be filtered to STR-friendly municipalities up front.

### Budget calibration
- Budget range is self-reported; cross-check against pre-approval before promising anything.
- If they pick a higher tier than their pre-approval supports, talk monthly payment, not price.
- Sub-$400k requires a tight neighborhood plan; supply is thin and moves fast.

### Reading the results page together
- This is the natural pause point for the next-steps conversation.
- Walk through each summary tile and confirm it matches what they meant.
- Use the Neighborhood and Strategy cards as the bridge to the rest of the meeting.
- If lifestyle says urban but commute says 45+ minutes, address the contradiction now.

### Follow-up plan from the quiz
- Confirm you will send the matched-MLS search by end of day.
- Calendar the rent vs buy walkthrough or the pre-approval intro within 48 hours.
- Send the bonus pack only after the conversation lands; it is leverage, not a giveaway.

---

## /neighborhoods

### How to use the county tabs
- These tabs are starting points, not boundaries. Most buyers end up in two adjacent counties.
- Anchor on commute time first; price tier and vibe follow once they see the map in their head.
- Median price is a vibe gauge, not a budget. Always pair with the active inventory you have in hand.
- Use the Watch Out For list as a credibility moment; it shows you flag risk, not just upside.

---

## /strategy

### Setting up the strategy conversation
- Use this section to make the offer playbook concrete, not theoretical.
- Start by asking the buyer which market they think we are in today.
- Their answer tells you how much education the offer conversation needs.
- Frame the playbook as the menu; the chosen tactics depend on the specific house.

### Seller's market notes
- Due Diligence Fee is the biggest signal of seriousness in NC; coach the buyer on a comfortable upper bound before tour day.
- Rent-back / possession after closing is underused. Ask the listing agent before you write.
- Trusted local lender beats big-bank prequal letters; remind the buyer their pre-approval letter is part of the offer.

### Balanced market notes
- Inspections come back to the negotiation table; pre-frame that expectation before you write the offer.
- Concessions toward a rate buydown often beat a price reduction on the same dollars.
- Use days-on-market data live in the meeting so the strategy is anchored in current inventory.

### Buyer's market notes
- Home sale contingencies become viable again; gauge the buyer's appetite to use them.
- Permanent rate buydowns through seller concessions can move monthly payment more than a price cut.

### Closing the strategy conversation
- Confirm the buyer can articulate the playbook back to you in one sentence.
- Schedule the next tour or offer-write session with a specific date and time on the calendar.
- Reinforce that every transaction is unique; the playbook gets edited live based on the property.

---

## /checklist

### Working the checklists together
- Use the three tabs to set expectations across the full buyer journey.
- Preparation tab is where most timelines slip; pre-approval and the budget conversation are non-negotiable.
- Hunting tab is the moment to introduce the MLS alert and the drive-through coaching.
- Closing tab keeps the buyer calm under contract; walk through it the day the offer is accepted.

---

## Review status

- Initial inventory: 2026-05-19 (this file). No content has been reviewed for accuracy yet — flag any line above that needs rework with a `🔴 REVIEW:` prefix.

## How to add or edit notes

Notes live in the Buyers Guide repo at `HGPG1/charlotte-buyers-guide`, inside the `AdvisorNote` components rendered on each buyer-facing page. Each section header is the prop title; each bullet is a child `<AdvisorNote>` text. Adding a note: edit the matching component, ship via the brain commit API or local push, then update this file to mirror the change.
