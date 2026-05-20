<!-- Last Updated: 2026-05-20 -->

# drafts/

Holding pen for brain content that's been authored by a Claude session but not yet merged into the canonical file (most often `SESSION-HANDOFF.md`).

## Why this folder exists

A Claude session that ships work needs to append a session entry to the TOP of `SESSION-HANDOFF.md` (append-only at the top, never overwrite prior entries). The safe write path is:

1. GET the full current `SESSION-HANDOFF.md` content via brain-app `/api/external/read?repo=hgpg-context&path=SESSION-HANDOFF.md`
2. Prepend the new entry under the `<!-- Last Updated -->` header
3. POST the full merged file back via `/api/external/write`

If step 1 fails (no token in scope, network hiccup, partial read), drop the new entry here as `drafts/YYYY-MM-DD-<topic>.md` so it survives the session end. Brian (or the next Claude session) merges it into `SESSION-HANDOFF.md` later.

## Conventions

- File naming: `YYYY-MM-DD-<short-topic>.md` (e.g. `2026-05-20-PM-idxre-b2-activation.md`)
- One entry per file. No multi-day grab-bags.
- Once merged into `SESSION-HANDOFF.md`, delete the draft via `gh rm` from Brian's Mac (the brain write API does not support delete).
