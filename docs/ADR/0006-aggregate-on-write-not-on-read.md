# ADR 0006: Aggregate on write, not on read

**Status:** Accepted
**Date:** 2026-05-27

## Context

FF15's UI exposes aggregated statistics: tier lists, win/pick/ban rates per
(champion, role, patch, region, rank bracket), build recommendations derived
from frequency analysis over millions of participant rows.

The raw data table (`MatchParticipant`) is expected to hold 2–3 million rows
per region at steady state. Tier list queries group across this table on
multiple dimensions.

Two architectural patterns are available:

1. **Aggregate on read** — UI requests trigger SQL aggregation queries against
   the raw table, with results cached for subsequent requests.
2. **Aggregate on write** — a scheduled job pre-computes aggregations into a
   purpose-built table, and UI requests read from that table directly.

## Decision

We will aggregate on write. A scheduled `stats:rebuild-daily` command will run
nightly, computing the `ChampionStatsDaily` table from raw participants via
raw DBAL SQL. UI controllers will read exclusively from `ChampionStatsDaily`
and never from `MatchParticipant`.

Build recommendations will follow the same pattern via a `builds:compute`
command writing to `BuildRecommendation`.

The web tier is read-only on the analytics tables. All writes happen in
scheduled worker jobs.

## Consequences

**Positive:**

- UI queries are single indexed SELECTs against a small pre-shaped table.
  Tier list rendering target: ~50ms including Twig render.
- The expensive aggregation work happens once per day, not once per user
  request.
- Aggregation logic lives in one place (a console command), not scattered
  across controllers.
- Cache becomes a latency optimization rather than a correctness requirement
  — the system works (slowly) even with a cold cache, because the underlying
  query is already fast.
- Web tier capacity planning is decoupled from analytics complexity.

**Negative:**

- Data is up to 24 hours stale. Acceptable for meta analytics — the meta
  doesn't shift hour-to-hour. Documented in the UI ("data refreshed daily").
- Adding a new aggregation dimension requires a schema change and a
  re-computation pass, not just a new query. This is a feature, not a bug:
  it forces deliberate decisions about what dimensions are exposed.
- Recomputation must complete within its window. Mitigated by scheduling
  during low-traffic hours and by the distributed lock preventing concurrent
  runs across worker pods.

## Rejected alternatives

**Aggregate on read with caching.** Rejected because:

- A cold cache means a multi-second response time, which is unacceptable for
  what should feel like an instant page load.
- Cache invalidation on data updates becomes more complex (every match insert
  potentially invalidates many cache keys).
- The expensive query gets re-run whenever any user hits a cache miss, which
  in the presence of varied filter combinations means it runs often.

**Materialized views.** A PostgreSQL-native alternative that achieves the
same goal. Rejected in favor of an explicit table for two reasons:

- The aggregation logic is non-trivial (tier bucketing, sample size filters,
  rank derivation) and is clearer expressed as a console command in PHP than
  as SQL embedded in a `CREATE MATERIALIZED VIEW` statement.
- Explicit tables are easier to migrate, test, and reason about than
  materialized views in a Symfony/Doctrine context.

This alternative could be revisited if the aggregation logic ever becomes
simple enough to fit naturally as a view.

## Future review

If users start asking for near-real-time aggregations ("how did this patch's
first hour of data look?"), this ADR should be revisited. Options would
include increasing recomputation frequency (hourly), streaming aggregation
(materialized view with incremental refresh, or an event-sourced approach),
or a hybrid where recent data is aggregated on read and historical data is
pre-computed.