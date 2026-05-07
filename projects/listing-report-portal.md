<!-- Last Updated: 2026-05-07 -->

# Listing Report Portal

- **Status:** 🟢 Live, GitHub auth blocker resolved
- **URL:** reports.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-listing-report
- **Supabase:** `wdheejgmrqzqxvgjvfee` (HGPG Listing Reports + MLS)
- **Vercel project ID:** in memory

## Recent

- 2026-05-06: Favicon push verified GitHub auth working (resolves prior blocker)

## Pending fix script

`~/Downloads/fix-admin-page.sh` may still have unpushed changes:

1. `SocialPostsManager` dark navy headings
2. `MagicLinkCard` Regenerate button placement
3. Admin section reorder
4. Zero-value stat hiding

Verify status next time portal is touched.

## Notes

- Supabase project shared with CMA Engine (MLS Grid replication tables)
- Database size at ~15GB (soft warning from Supabase). Pruning plan exists, "revisit" status. Plan: prune MLS data older than 36 months + drop unused indexes. Estimated reclaim 8-10 GB.
