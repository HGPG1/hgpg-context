# CMA Engine

- **URL:** cma.homegrownpropertygroup.com
- **Repo:** HGPG1/hgpg-cma-tool
- **Vercel:** prj_6kVABGRQakKQ36G1xjPtxH5ZztwN
- **Supabase:** ioypqogunwsoucgsnmla
- **PIN:** 2026

## Architecture decisions

### Status-weighted Perceived Market Value (PMV)

- Solds: **60%**
- Under Contract - No Showing pendings: **25%**
- Under Contract - Showing pendings: **10%**
- Actives: **5%**

### Sharran Srivatsaa three-strategy pricing framework

| Strategy | Price | Notes |
|---|---|---|
| Aspirational | PMV + 3-7% | |
| PMV anchor | PMV | |
| Event | PMV - 2-5% | |

- Verbatim scripts required.
- **Sharran attribution required** in output.

### Hard rule

**Never use "fair market value."** Always use "Perceived Market Value" / PMV.

## Data sources

- **Canopy MLS:** UC-Show / UC-NoShow sub-status logic wired in.
- **IDX Broker `/mls/search` endpoint:** NOT enabled on current tier.
- **Manual comp entry:** v1 primary input path until IDX endpoint is enabled.

## April 19, 2026 build session

Seven commits shipped:

1. GLA methodology fix
2. Neighborhood bracketing
3. Duplicate row bug fix on reopen-and-regen
4. History filter
5. DOM fix
6. Methodology doc comments
7. Renamed "Value-Defense Report" → "Supporting Market Analysis"
