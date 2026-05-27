# ADR 0007: Link Data Dragon images from Riot's CDN; do not rehost

**Status:** Accepted
**Date:** 2026-05-27

## Context

FF15 displays significant amounts of static game imagery: champion portraits,
item icons, rune icons, summoner spell icons, profile icons. All of this
content is hosted by Riot on Data Dragon's CDN, versioned per patch, freely
accessible without authentication or rate limiting.

A choice must be made between:

1. Linking directly to Data Dragon URLs in HTML.
2. Downloading images during the catalogue sync and serving them from
   FF15's own infrastructure.

## Decision

We will link directly to Data Dragon CDN URLs from rendered HTML.

For each catalogued entity (`Champion`, `Item`, etc.), we will store only the
image filename string from Data Dragon (e.g., `Ahri.png`) in the database. A
Twig helper or asset host configuration will resolve this to the full CDN URL
at render time, using the appropriate patch version.

## Consequences

**Positive:**

- Zero storage cost for images. The catalogue sync deals with JSON metadata
  only — a few megabytes per patch.
- Zero bandwidth cost for serving images to users. Riot's CDN handles it.
- Riot's CDN is global and fast; ours would not be.
- Old patches remain accessible indefinitely — Data Dragon preserves every
  historical version, supporting historical analysis views without any
  additional work.
- New patches add new images automatically; nothing breaks when a champion or
  item is added.
- No risk of stale image cache on our side when Riot ships a visual update.

**Negative:**

- Hard dependency on Data Dragon's availability. If Riot's CDN is down, our
  pages render without images. Acceptable for a portfolio project; would be
  worth mitigating in a commercial product.
- Cannot apply our own image transformations (resizing, format conversion,
  WebP delivery). We are at the mercy of Data Dragon's formats and sizes.
  Acceptable — the formats are fine.
- Content Security Policy must allow image loads from `ddragon.leagueoflegends.com`.

## Rejected alternatives

**Download images during catalogue sync, serve from our infrastructure.**
Rejected because:

- It adds storage cost (the full image set is ~1 GB per patch via the
  `dragontail` tarball; selective download is smaller but still significant
  across many patches).
- It adds bandwidth cost.
- It adds a failure mode (sync job failed → users see broken images for new
  champions).
- Riot's CDN will deliver these images faster than ours will, from more edge
  locations.
- There is no scenario in this project where rehosting provides a benefit.

**Download images on demand, cache locally.** Rejected for the same reasons,
plus the added complexity of an on-demand fetching layer.

## Future review

Revisit if Riot ever deprecates Data Dragon, changes its terms to forbid
hotlinking, or if FF15 develops requirements that demand image transformations
not possible against the upstream CDN (e.g., a "compare two champions side by
side" view that needs identically-sized portraits).

The catalogue sync stores the filename strings independently of any URL
template, so switching to a self-hosted image strategy in the future would
require only adding the download step and changing the URL resolution helper —
not a database migration.