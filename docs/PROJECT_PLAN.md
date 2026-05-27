# FF15 — Project Plan

> A fullstack League of Legends data project showcasing modern Symfony development
> and a production-style Kubernetes deployment. Two pillars: champion/meta analytics
> and build optimization, derived from aggregated high-elo match data.

---

## Table of contents

1. [Goals & scope](#1-goals--scope)
2. [Stack decisions](#2-stack-decisions)
3. [Component inventory](#3-component-inventory)
4. [Global architecture](#4-global-architecture)
5. [Data sources](#5-data-sources)
6. [Data fetching strategy](#6-data-fetching-strategy)
7. [Domain model](#7-domain-model)
8. [Redis role map](#8-redis-role-map)
9. [Delivery phases](#9-delivery-phases)
10. [Things to deliberately NOT do](#10-things-to-deliberately-not-do)
11. [Future extensions](#11-future-extensions)

---

## 1. Goals & scope

A two-pillar LoL data application built as a portfolio piece, showcasing both
development depth (Symfony, async pipelines, real analytics) and operations
depth (Kubernetes on RKE2, scaling, observability).

### Pillars

- **Pillar A — Champion & meta analytics.** Tier lists, win/pick/ban rates by
  patch, role, rank, region. Matchup matrices. Patch-over-patch trend lines.
- **Pillar B — Build optimizer.** Recommended items, runes, skill order for a
  champion in a given role, optionally against a specific opponent. Backed by
  real aggregated data with confidence indicators (sample size, winrate delta).
- **Pillar C (deferred).** Summoner lookup that ties personal play into A and B.

### Non-goals (v1)

- Real-time match tracking or live-game features
- User accounts, saved comparisons, or social features
- Mobile app or SPA frontend
- Public REST/GraphQL API surface

---

## 2. Stack decisions

| Layer | Choice | Reasoning |
|---|---|---|
| Framework | **Symfony 8.0** | Current stable. Modern over LTS for a portfolio piece — recruiters notice. |
| Language | **PHP 8.4** | Required by Symfony 8. Property hooks, asymmetric visibility, readonly classes. |
| Templating | **Twig** | First-class Symfony integration. |
| Interactivity | **Symfony UX Live Components + Stimulus + Turbo** | Server-driven UI without SPA complexity. |
| Styling | **Tailwind CSS** (via `symfonycasts/tailwind-bundle`) | Integrates with AssetMapper. |
| Database | **PostgreSQL 16** | `jsonb` for raw Riot payloads alongside normalized columns. |
| Cache / queue / coordination | **Redis 7+** | Five distinct roles, logically separated by DB index. |
| Orchestration | **RKE2 Kubernetes** | DevOps pillar of the portfolio. |
| CI | **GitHub Actions** | PHPUnit, PHPStan level 8, php-cs-fixer, Twig lint. |

---

## 3. Component inventory

Every piece of the stack, what it does, and who talks to it.

### Application layer

- **Symfony 8.0** — the monolith framework. One codebase, three runtime roles
  (web, worker, scheduler). Hosts controllers, services, Doctrine ORM, console
  commands, Messenger handlers.
- **PHP 8.4** — runtime. Required by Symfony 8.
- **Twig** — server-side templating. Used by controllers and Live Components.
- **Symfony UX Live Components** — server-driven interactivity. Filter bars,
  selectors, admin dashboards re-render over AJAX without an SPA.
- **Stimulus** — thin client-side JS for things that must happen in the browser
  (chart rendering, sortable tables, clipboard).
- **Turbo** — partial page swaps, Turbo Frames for slide-out drawers.
- **Tailwind CSS** — styling, integrated via AssetMapper.

### Data layer

- **PostgreSQL 16** — system of record. Matches, participants, summoners,
  patches, static catalogue (champions/items/runes), pre-computed aggregations,
  build recommendations. `jsonb` columns hold raw Riot payloads.
- **Doctrine ORM + DBAL** — ORM for entity CRUD, raw DBAL SQL for analytics
  queries (window functions, GROUP BY across millions of rows).

### Redis (five roles, separate DB indexes)

See [Redis role map](#8-redis-role-map) for full details.

- **DB 0** — Messenger transport (job queues, Redis Streams)
- **DB 1** — Rate limiter state (Riot API budget)
- **DB 2** — Deduplication & cooldowns
- **DB 3** — Application cache (tagged, with eviction)
- **DB 4** — Distributed locks (single-leader for scheduled jobs)

### External services

- **Riot Games API** — dynamic data (matches, summoners, leaderboards).
  Rate-limited, requires API key, accessed only by workers.
- **Data Dragon** — Riot's free CDN of static data (champions, items, runes,
  images), versioned per patch. See [Data sources](#5-data-sources).

### Operational

- **Messenger workers** — long-running PHP processes consuming Redis streams.
  Scaled horizontally based on queue depth.
- **Symfony Scheduler** — single-replica process dispatching scheduled
  messages. Replaces cron.
- **GitHub Actions** — CI pipeline.

---

## 4. Global architecture

**Three runtimes, one codebase, shared state in Postgres + Redis.**

```
                          ┌──────────────────┐
                          │   Browser (user) │
                          └────────┬─────────┘
                                   │ HTTPS
                                   ▼
                          ┌──────────────────┐
                          │   Web tier       │
                          │   (Symfony +     │
                          │    Twig + Live   │
                          │    Components)   │
                          │                  │
                          │   READ ONLY      │
                          │   from DB        │
                          └────────┬─────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
       ┌────────────┐       ┌────────────┐      ┌────────────┐
       │ PostgreSQL │       │   Redis    │      │  (no Riot  │
       │            │       │  DB 3:     │      │   calls    │
       │  aggreg.   │       │  cache     │      │   here)    │
       │  tables    │       │            │      │            │
       └─────▲──────┘       └─────▲──────┘      └────────────┘
             │                    │
             │ writes             │ invalidates
             │                    │
       ┌─────┴────────────────────┴──────┐
       │     Worker tier (Messenger)     │
       │                                 │
       │  - FetchMatchHistoryHandler     │
       │  - FetchMatchHandler            │
       │  - RecomputeStatsHandler        │
       │  - ComputeBuildsHandler         │
       │  - SyncDataDragonHandler        │
       │                                 │
       │  Horizontally scaled            │
       └─────────┬───────────────┬───────┘
                 │               │
                 ▼               ▼
          ┌─────────────┐  ┌──────────────┐
          │   Redis     │  │  Riot API    │
          │   DB 0:     │  │              │
          │   queues    │  │  (rate       │
          │   DB 1:     │  │   limited)   │
          │   limiter   │  │              │
          │   DB 2:     │  │  Data        │
          │   dedup     │  │  Dragon      │
          │   DB 4:     │  │              │
          │   locks     │  │              │
          └─────────────┘  └──────────────┘
                 ▲
                 │ dispatches messages
                 │
       ┌─────────┴─────────┐
       │  Scheduler tier   │
       │  (Symfony         │
       │   Scheduler)      │
       │                   │
       │  Single replica   │
       └───────────────────┘
```

### The key architectural rule

**The web tier never calls Riot. The web tier never writes to analytics tables.**

Web tier reads from Postgres (aggregations, catalogue) and Redis DB 3 (cache).
All Riot API calls and all writes to analytics tables happen in workers,
asynchronously. User-facing pages have no upstream dependency in the request path.

### Three runtimes from one codebase

- **Web pods** — `php-fpm` behind nginx. Scaled by HTTP load.
- **Worker pods** — `bin/console messenger:consume ingestion_high ingestion_matches aggregations`. Scaled by queue depth.
- **Scheduler pod** — `bin/console messenger:consume scheduler_default`. Single
  replica — there must be exactly one source of scheduled events.

Same Docker image, different command. Maps to three Deployments on RKE2.

### Data flow

- **Read path** (user → tier list page): browser → web pod → Postgres
  aggregation table (or Redis cache hit) → Twig render → response. Target ~50ms.
- **Write path** (Riot → aggregation): scheduler → Messenger queue → worker →
  Riot API → Postgres raw tables → worker → Postgres aggregations → cache
  invalidation. Asynchronous, can take hours, fully decoupled from user requests.

---

## 5. Data sources

### Riot Games API

The dynamic source. Two API surfaces:

- **Platform routes** (per-region: EUW1, NA1, KR) — summoner, league endpoints
- **Regional routes** (per-cluster: europe, americas, asia) — match-v5 endpoints

Constraints that shape the architecture:

- Two-layer rate limiting (app limit + per-method limit, both per-region)
- Dev keys regenerate every 24h; production keys require an application
- No bulk-query endpoints — match corpus must be built incrementally from real
  players outward

### Data Dragon (ddragon)

Riot's free, unauthenticated CDN for **static** game data — champions, items,
runes, summoner spells, profile icons, plus all images, snapshotted per patch.

- No API key, no rate limit
- Versioned per patch (e.g., `14.10.1`); old versions preserved forever
- Updates lag the actual game patch by 0–2 days

**URL structure:**

- Discovery: `https://ddragon.leagueoflegends.com/api/versions.json` (flat
  array, newest first), `/realms/{region}.json` (which version is live where)
- Data: `https://ddragon.leagueoflegends.com/cdn/{version}/data/{locale}/{file}.json`
  (champion.json, item.json, runesReforged.json, summoner.json, etc.)
- Images: `https://ddragon.leagueoflegends.com/cdn/{version}/img/{type}/{filename}`

**How this project uses it:**

- Scheduled job every 24h: fetch `versions.json`, compare to latest `Patch` row,
  ingest new patch data if changed. Idempotent.
- Store identifying/queryable fields normalized in Postgres + the full ddragon
  entry as `jsonb` for flexibility.
- **Link images directly from ddragon's CDN** — don't rehost. Store the image
  filename string, build URLs in Twig.

**What ddragon does NOT provide** (and where to get it):

- Win/pick/ban rates → computed by our Pillar A pipeline from match data
- Patch release dates → fill manually or from Community Dragon
- Reliable spell variable resolution → use Community Dragon if needed

### Community Dragon (cdragon, optional)

Community-maintained project extracting from game files. Not used in v1; potential
fallback for patch dates if needed later.

---

## 6. Data fetching strategy

### Fundamental constraint

Riot's API does not let you query "all matches from patch X in region Y."
The corpus is built incrementally: seed from leaderboards → walk match histories
→ fetch match details. After ~48h of continuous ingestion, a representative
sample of the current patch exists.

### Three-stage pipeline

**Stage 1 — Seeding.** Scheduler dispatches `SeedHighEloMessage` every 6h per
region. Handler fetches challenger/grandmaster/master leagues, upserts summoners,
dispatches `FetchMatchHistoryMessage` for each.

**Stage 2 — Match ID discovery.** `FetchMatchHistoryMessage` handler checks the
summoner's cooldown key in Redis DB 2. If present, skip. Otherwise call Riot's
match-history endpoint, set cooldown (6h TTL), and for each new match ID try to
claim an in-flight lock (Redis SET NX EX 1h). If claimed, dispatch
`FetchMatchMessage`; if not, another worker has it.

**Stage 3 — Match detail fetch.** `FetchMatchMessage` handler calls Riot's match
endpoint, validates (queue 420 = ranked solo, ignore others), and in a single
Postgres transaction inserts the `Match` row and 10 `MatchParticipant` rows.
Raw JSON goes into `Match.raw` (jsonb). Unique constraint on `riot_match_id` is
the safety net against any duplicates that slip past the dedup layer.

### Why three stages

Each stage's failure mode is different. Decomposition means each unit of work is
idempotent and retryable in isolation. A failed match detail fetch doesn't lose
the match ID; a failed history fetch doesn't lose the summoner.

### Rate limiting in practice

Every outbound Riot call goes through a `RiotApiClient` wrapper that consumes
from two limiters before the HTTP request:

- Method limiter: `riot_match_{region}` (sliding window, 80% of Riot's limit)
- App limiter: `riot_app_{region}` (sliding window, 80% of Riot's limit)

Workers use `reserve()->wait()` — block until tokens are available rather than
throwing. Headroom of 20% absorbs 429 retries and burst events. When a 429 does
slip through (Riot's own edge-side limiting), honor `Retry-After`, nack with
delay, let Messenger retry strategy take over.

### Static data fetching (Data Dragon)

Separate, simpler path. `SyncDataDragonMessage` runs nightly:

1. Fetch `versions.json`, compare to latest `Patch` row
2. If new version, fetch `championFull.json`, `item.json`, `runesReforged.json`,
   `summoner.json`
3. Upsert into Postgres
4. Invalidate cache tags for the previous patch

No rate limit, no dedup needed.

### Aggregation

Web tier never reads `MatchParticipant` directly — it reads `ChampionStatsDaily`,
computed nightly by `RecomputeChampionStatsMessage`. Handler acquires a lock in
Redis DB 4 (prevents concurrent runs across pods), runs raw DBAL SQL with GROUP
BY (champion, role, patch, tier_bucket, region) plus window functions, upserts
into `ChampionStatsDaily`.

Build recommendations follow the same shape (`ComputeBuildsMessage`): SQL
aggregation of common item orderings per (champion, role, opponent) where sample
size ≥ 100 and winrate > median.

### Cleanup

`CleanupOldMatchesMessage` runs weekly: deletes `Match` and `MatchParticipant`
rows older than the last 3 patches. Aggregations preserved. Keeps raw tables
bounded at steady state (~2-3M participant rows per region).

---

## 7. Domain model

Doctrine entities, sketched.

- **Patch** — `version`, `releasedAt`, `isCurrent`. Anchors everything.
- **Champion** — synced from Data Dragon per patch. `riotId`, `key`, `name`,
  `roles[]`, `tags[]`, `image_full`, `raw` (jsonb), `patch_id` (FK).
- **Item / Rune / SummonerSpell** — same pattern, Data Dragon synced.
- **Match** — `riotMatchId` (unique), `region`, `patch_id`, `queueId`,
  `gameDuration`, `gameCreation`, `raw` (jsonb).
- **MatchParticipant** — denormalized for query speed. `match_id`, `puuid`,
  `champion_id`, `teamPosition`, `win`, KDA, items[6], runes, summoner spells,
  goldEarned, damageDealt, cs. The analytics workhorse table.
- **Summoner** — `puuid`, `gameName`, `tagLine`, `region`, `currentTier`,
  `lastFetchedAt`.
- **ChampionStatsDaily** (materialized aggregation) — `champion_id`, `patch_id`,
  `role`, `tier_bucket`, `region`, `date`, `picks`, `wins`, `bans`, `avg_kda`.
  **Web tier reads this, never the raw participants table.**
- **BuildRecommendation** — pre-computed per (champion, role, patch, opponent).
  Ordered item sequence + sample size + winrate.

Pattern: raw data → denormalized participants → nightly aggregations → UI reads
aggregations.

---

## 8. Redis role map

One Redis instance, five logical DBs.

### DB 0 — Messenger transport (queues)

Redis Streams via Symfony Messenger. Three streams with different rate profiles:

- `ingestion_high` — leaderboard seeding, low volume
- `ingestion_matches` — the firehose, thousands during burst seeding
- `aggregations` — nightly compute jobs

Separating streams prevents a slow handler in one from blocking another.

Failed messages → Doctrine table (not Redis) for durability across restarts.

Config: `delete_after_ack: true`, `delete_after_reject: true`. Cap with
`MAXLEN ~ 100000` as safety net.

### DB 1 — Rate limiter state

Symfony RateLimiter with Redis storage backend. Shared budget across all
worker pods.

- Sliding window policy (closer to Riot's actual enforcement than fixed window)
- Per-method per-region limiters (`riot_match_{region}`, `riot_summoner_{region}`)
- Per-region app limiter (`riot_app_{region}`)
- All limits set at 80% of Riot's published values

Every Riot call consumes from both the method limiter and the app limiter.
Workers `reserve()->wait()`.

### DB 2 — Deduplication & cooldowns

Prevents redundant work in the ingestion pipeline.

- `inflight:match:{id}` — SET NX EX 3600. Stops the same match being enqueued
  multiple times when many summoners played together.
- `cooldown:summoner:{puuid}` — EX 21600. Prevents refetching a summoner's
  history within 6h.

Pairs with the DB-level unique constraint on `match.riot_match_id` for defense
in depth.

### DB 3 — Application cache

Tagged cache via `RedisTagAwareAdapter`. Pools:

- `riot.cache` — 1h default TTL (rendered tier lists, build queries)
- `riot.cache.long` — 24h default TTL (Data Dragon manifests)

Tags enable bulk invalidation: `$cache->invalidateTags(['patch:14_10'])` when
new aggregations land.

**Only DB where eviction is acceptable.** `allkeys-lru` policy if memory pressure.

### DB 4 — Distributed locks

Symfony Lock with Redis store. Ensures single-leader semantics for scheduled
jobs when worker tier scales.

- `aggregation:champion-stats` — only one pod recomputes stats
- `aggregation:builds` — only one pod recomputes builds
- `datadragon:sync` — only one pod syncs catalogue

### Persistence config

- **AOF** with `appendfsync everysec` — acceptable to lose 1s of writes, not
  acceptable to lose the queue.
- **noeviction** globally (queues, rate limit, dedup, locks must not be evicted),
  with aggressive TTLs on cache keys to manage memory naturally.

---

## 9. Delivery phases

Each phase is independently shippable.

### Phase 1 — Foundations (week 1)

- `symfony new --webapp` with Symfony 8 + Twig + Doctrine + Forms + AssetMapper
- Install UX Live Components, UX Twig Component, UX Turbo, Messenger, RateLimiter,
  Tailwind bundle, Doctrine Migrations
- Docker Compose for local dev (php-fpm, nginx, postgres, redis, mailpit)
- `.env` + Symfony secrets vault for `RIOT_API_KEY`
- GitHub Actions: PHPUnit, PHPStan level 8, php-cs-fixer, Twig lint
- Tailwind theme — restrained LoL-ish dark palette, real type system

**Showcases:** clean project setup, modern PHP, CI hygiene.

### Phase 2 — Riot API client + Data Dragon sync (week 2)

- Custom HTTP client on Symfony HttpClient with RateLimiter (Redis-backed),
  retry on 429/503 honoring `Retry-After`, PSR-3 logging
- `RegionRouter` service abstracting platform vs regional routes
- `riot:datadragon:sync` console command, idempotent, runs on deploy + nightly

**Showcases:** HttpClient, RateLimiter, console commands, DI.

### Phase 3 — Match ingestion pipeline (weeks 3-4)

- `riot:seed:high-elo` command — fetch leaderboards, dispatch history fetches
- `FetchMatchHistoryMessage` handler with cooldown checks
- `FetchMatchMessage` handler with transactional persistence
- Symfony Scheduler — seed every 6h, cleanup weekly
- Health endpoint exposing queue depth + last successful fetch per region

**Showcases:** Messenger, async workflows, Scheduler, transactional consistency,
idempotency design.

### Phase 4 — Aggregations + Pillar A UI (weeks 5-6)

- `stats:rebuild-daily` command (DBAL raw SQL, distributed lock)
- Tier list page `/champions` with Live Component filter bar
- Sortable columns via Stimulus
- Turbo Frame for champion detail drawer
- Champion detail page `/champion/{slug}` — synergies, counters, trend lines
- Patch-diff page `/patch/{from}/vs/{to}`

**Showcases:** Twig + Live Components + Stimulus + Turbo, raw SQL analytics,
thoughtful UX.

### Phase 5 — Pillar B: Build optimizer (weeks 7-8)

- `builds:compute` command — frequency analysis of item paths, runes, skill orders
- `/champion/{slug}/builds` with "against" selector (Live Component)
- Confidence indicators on every recommendation (sample size, winrate delta)

**Showcases:** real analytics, statistical rigor, polished interactive UI.

---

## 10. Things to deliberately NOT do

- **Don't store all matches forever.** Cap at last 3 patches per region.
  `cleanup:old-matches` runs weekly.
- **Don't compute aggregations on read.** Always pre-compute. Tier list page
  is a single indexed query against `ChampionStatsDaily`.
- **Don't fight Riot's rate limits.** Build at 80% of limit for headroom.
- **Don't put Riot calls in the request/response path.** All Riot fetches in
  workers or commands. UI reads from local DB only.
- **Don't rehost Data Dragon images.** Link to Riot's CDN.
- **Don't use API Platform in v1.** No API consumer exists. Defer to extensions.
- **Don't add Mercure.** Live Components + polling covers every freshness need
  in this project's scope.

---

## 11. Future extensions

Post-v1, in rough order of value:

- **Pillar C — Summoner lookup.** New ingestion path + comparison view tying
  personal play into A and B.
- **API Platform layer.** Public read-only REST/GraphQL over the aggregated
  data, with auto-generated OpenAPI docs.
- **Auth.** Magic-link login or Discord OAuth ("sign in with Discord" is
  on-brand). Saved comparisons, favorite champions.
- **Patch release dates from Community Dragon.** For accurate time-axis charts.
- **Multi-region aggregation views.** Currently per-region; cross-region
  comparison would be a nice analytical view.

---

## Document conventions

- Architecture decisions are recorded inline; significant changes should be
  added with date + rationale.
- Code snippets and implementation details belong in `docs/` subdocuments, not
  here. This document is the high-level map.