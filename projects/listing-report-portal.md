# Listing Report Portal

- **URL:** reports.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-listing-report
- **Supabase + Vercel project IDs:** in memory

## Pending fix script

`~/Downloads/fix-admin-page.sh` has **4 unpushed changes:**

1. `SocialPostsManager` dark navy headings
2. `MagicLinkCard` Regenerate button placement
3. Admin section reorder
4. Zero-value stat hiding

## Blocker

**GitHub auth not configured on Mac Mini.** Resolution paths:

1. Set up SSH keys on Mac Mini
2. Working PAT on Mac Mini
3. Use Supabase `github-push` Edge Function
4. Use GitHub web API from authenticated browser session
