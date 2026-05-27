# ADR 0004: Five-role separation of Redis usage

**Status:** Accepted
**Date:** 2026-05-27

## Context

Redis appears in multiple distinct roles in the FF15 architecture: as the
Messenger transport, as the rate limiter storage, as a deduplication and
cooldown layer, as the application cache, and as the distributed lock
backend.

These roles have different durability, eviction, and sizing requirements.
The naive approach — treat Redis as a single bucket and dump everything in —
creates correctness risks (evicting an in-flight job, flushing the rate
limiter during a cache incident) and operational opacity (no way to tell which
role is consuming memory).

A choice must be made about how to organize these roles operationally.

## Decision

We will run a single physical Redis instance with five **logically separated**
DB indexes, one per role:

| DB | Role | Eviction policy | Persistence requirement |
|----|------|------------------|--------------------------|
| 0 | Messenger transport (Redis Streams) | None — must not evict | Durable (AOF) |
| 1 | Rate limiter state | None — must not evict | Durable (AOF) |
| 2 | Deduplication & cooldowns | None — TTL only | Durable (AOF) |
| 3 | Application cache | LRU — eviction expected | Best-effort |
| 4 | Distributed locks | None — TTL only | Durable (AOF) |

Each Symfony component will be configured with its own DSN pointing at the
appropriate DB index.

The global Redis configuration will use `maxmemory-policy noeviction` and
`appendfsync everysec`. The cache DB (3) will manage memory via aggressive
TTLs on its keys rather than relying on eviction.

## Consequences

**Positive:**

- A `FLUSHDB` during a cache incident wipes only the cache, not the queues or
  rate limit state.
- Memory usage is observable per role via `INFO keyspace`.
- Operational mental model is clear — each DB has one owner and one purpose.
- Adding or removing a role doesn't disturb the others.
- Documentation and ADRs can refer unambiguously to "the queue DB" or "the
  cache DB."

**Negative:**

- Single Redis instance is a single point of failure. Acceptable for a
  portfolio project; would need clustering or sentinel for true production.
- All roles share the same memory budget. The `noeviction` policy means that
  if the cache DB pushes memory toward the limit, writes to other DBs could
  fail. Mitigated by aggressive cache TTLs and bounded stream sizes.
- Redis DBs are a feature some operators discourage in favor of fully separate
  instances. For this project's scale, the operational simplicity wins.

## Rejected alternatives

**Single Redis, no separation (all roles share DB 0).** Rejected because of
the correctness and operational risks described above.

**Separate Redis instances per role.** Rejected as premature optimization at
this scale. Five instances multiplies operational overhead (five
StatefulSets, five backup strategies, five sets of credentials) without
proportional benefit. A future ADR can split out the cache DB onto its own
instance if memory pressure becomes a real problem.

**Single Redis with key prefixes instead of DB indexes.** Rejected because
prefix-based separation doesn't give per-role flush, per-role memory
observability, or per-role eviction policy. DB indexes are the right tool for
this job.

## Future review

If memory pressure between cache (DB 3) and the no-eviction DBs becomes an
operational issue, split the cache onto a separate Redis instance with an
appropriate eviction policy (`allkeys-lru`). This is a one-config-change
operation.

If the project ever requires Redis high availability, this ADR should be
superseded by one proposing Sentinel or Redis Cluster.