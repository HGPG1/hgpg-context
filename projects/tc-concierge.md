# TC Concierge

- **URL:** tc.homegrownpropertygroup.com
- **Supabase:** ioypqogunwsoucgsnmla (contains `transaction_emails` table; will host Phase 2 deals table)
- **Status:** Live. API key fix complete. Don is running real deals through it.

## Google Apps Script TC ingestion

Both accounts tuned to:

- **Hourly triggers**
- `newer_than:7d` lookback
- **Business hours gate:** Mon-Fri, 8am-6pm ET (prevents after-hours API calls)
