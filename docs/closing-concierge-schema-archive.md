# Closing Concierge (standalone) — schema archive

Archived 2026-05-05 when the standalone Closing Concierge product was decommissioned. The TM-integrated concierge auto-trigger (commit 1fd2d02, 2026-04-25) replaced it.

Original Supabase project: mvhyurdomejzdfigmyuk (closing-concierge)
Original domain: concierge.homegrownpropertygroup.com

Useful as a reference if the TM-integrated concierge ever needs to be enriched with smart-device, utility, or move-in/move-out detail capture. The shapes below were live in production-shaped form (1 test row) and are reasonable starting points.

## Tables

### properties
Core record per closing transaction.
- id: uuid (pk)
- created_at, updated_at: timestamptz
- address: text
- closing_date: date
- role: text (agent | seller | buyer)
- status: text (active | completed | cancelled)
- agent_name, agent_email, agent_phone, agent_brokerage, agent_logo_url, agent_brand_color: text
- seller_name, seller_email, seller_phone: text
- buyer_name, buyer_email, buyer_phone: text
- internet_provider, router_stays, modem_stays, router_brand, router_model, internet_notes: text
- smart_devices: jsonb (array of { brand, category, location, notes, connectedToApp, leavingWithHome })
- keypad_locks: jsonb ({ hasLocks: bool, locks: [{ type, brand, model, location, codesProgrammed }] })
- utilities: jsonb ({ gas, water, electric, recycling, yardWaste, trashIncluded, trashProvider, trashPickupDay })
- garage_access: jsonb ({ keypadPresent, vehiclePairing, remotesIncluded, smartOpenerBrand })
- mail_forwarding: jsonb ({ buyerReminder, sellerForwardingSet })
- buyer_guide_generated, seller_checklist_generated, agent_summary_generated, emails_sent, sms_scheduled: bool

### sms_schedule
Scheduled outbound SMS to seller/buyer/agent.
- id (pk), property_id (fk → properties), recipient_phone, recipient_type, message, scheduled_date, sent_at, sent, error

### email_log
Sent-email audit.
- id (pk), property_id (fk → properties), recipient_email, recipient_type, subject, status, sent_at

### pages
Generic landing page CMS, 0 rows in production. Slug + html.

## Categories captured for smart_devices
Smart locks, Thermostat, Garage opener, Doorbell, Alarm system, Smart switches, Smart irrigation. Each has location, brand, notes, app-connection state, and stays-with-home flag.

## What's worth porting if TM concierge gets richer

- The smart_devices jsonb shape is the most interesting — captures real handoff knowledge a buyer needs day one
- Utilities object is a clean breakdown for a move-in welcome packet
- Garage access detail (keypadPresent, vehiclePairing, remotesIncluded, smartOpenerBrand) is the kind of thing that gets forgotten at closing
