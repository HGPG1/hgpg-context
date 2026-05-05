# HGPG Team Tools

Last updated: 2026-05-05

## Live URLs
- Production: https://tools.homegrownpropertygroup.com
- Vercel project: hgpg-team-tools2

## Repo
- HGPG1/hgpg-team-tools2 (Vite + React + TypeScript + Tailwind)
- Branch: main
- Auto-deploy on push to main via Vercel

## Auth
- Provider: Google OAuth via Supabase
- Supabase project: ioypqogunwsoucgsnmla (primary)
- Workspace-domain enforcement: Internal OAuth client at Google layer + email regex check at app layer (defense in depth)
- Allowlist regex: /@homegrownpropertygroup\.com$/
- Google Cloud project: HGPG Team Tools (project ID hgpg-team-tools, project number 429376800551)
- OAuth client name: HGPG Team Tools Web Client
- Authorized JavaScript origins: tools.homegrownpropertygroup.com, ioypqogunwsoucgsnmla.supabase.co, localhost:5173
- Authorized redirect URIs: ioypqogunwsoucgsnmla.supabase.co/auth/v1/callback, localhost:5173/auth/callback

## Supabase config
- Site URL: https://tools.homegrownpropertygroup.com
- Redirect URLs: tools.homegrownpropertygroup.com/auth/callback, localhost:5173/auth/callback
- Tables (public schema):
  - agent_profiles (RLS: users read/insert/update own row only; no recursive admin policy)

## Vercel env vars
- VITE_SUPABASE_URL: https://ioypqogunwsoucgsnmla.supabase.co
- VITE_SUPABASE_ANON_KEY: anon JWT for the project (10y expiry)
- ANTHROPIC_API_KEY: powers /api/generate (flagged Needs Attention, low priority)
- FRED_API_KEY: powers /api/market-data (flagged Needs Attention, low priority)
- RENTCAST_API_KEY: legacy, may not be in use (flagged Needs Attention, low priority)

## Code shape
- src/lib/supabase.ts reads VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY from import.meta.env (NOT hardcoded)
- src/hooks/useAuth.ts handles signInWithGoogle, allowlist check, agent_profiles fetch
- src/pages/AuthCallback.tsx handles the post-Google redirect
- api/generate.ts proxies Claude Haiku for tool generation
- api/market-data.ts proxies FRED for Charlotte MSA market data
- No service role key anywhere in client or server code

## Known cleanup / open items
- ANTHROPIC_API_KEY, FRED_API_KEY, RENTCAST_API_KEY flagged "Needs Attention" in Vercel env vars panel (likely missing from one or more environments) - revisit if /api/generate or /api/market-data fails
- DNS Change Recommended on tools.homegrownpropertygroup.com domain (Vercel planned IP range expansion) - old CNAME continues to work, low priority

## What broke previously
- Supabase project nfwjgsanvmwmrginhklz was the original auth backend, deleted during Suna teardown 2026-05-04. Hardcoded URL in src/lib/supabase.ts orphaned the auth flow until env-var refactor + Google OAuth reset on 2026-05-05.
