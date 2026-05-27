# FF15 — Development Roadmap

A phased plan describing *what* gets done in each phase, not *how*. Each phase
produces something demonstrable.

---

## Phase 0 — Repository foundation

Set up the monorepo before writing any application code. Decisions made here
are hard to change later.

- Create the `ff15` repository with the top-level structure (`app/`, `infra/`, `docs/`)
- Write the initial README as the project's front door
- Establish the ADR practice — first ADR documents the monorepo decision itself
- Set up branch protection, PR templates, commit conventions
- Configure GitHub Actions skeleton with path filters separating app and infra workflows
- Decide on the development workflow: branching model, PR review approach

**Done when:** the repo exists, is publicly visible, has a meaningful README
and at least one ADR. Nothing builds yet, but the structure tells a story.

---

## Phase 1 — Local development environment

Get a working local stack before touching application logic. Friction here
compounds across every later phase.

- Docker Compose stack for local dev: PHP-FPM, nginx, PostgreSQL, Redis, mailpit
- Symfony project bootstrap with all required bundles installed
- Environment configuration strategy: `.env` layering, Symfony secrets vault, how secrets flow in CI and prod
- Makefile or task runner for the common commands you'll run hundreds of times
- Database connection working, first migration runs, Redis reachable
- Tailwind compilation working via AssetMapper
- Hot-reload loop confirmed (edit Twig → see change in browser)

**Done when:** `make up` from a fresh clone gets a developer to a working
Symfony welcome page in under two minutes.

---

## Phase 2 — Quality and CI gates

Establish the quality bar before there's enough code to make it painful to
retrofit.

- PHPUnit configured, first smoke test passing
- PHPStan at level 8 with a clean baseline
- php-cs-fixer with a documented ruleset
- Twig linting
- Doctrine schema validation in CI
- GitHub Actions workflows: app CI runs on `app/**` changes, infra linting runs on `infra/**` changes
- Pre-commit hooks (optional but recommended)
- Decide what blocks a merge vs. what's advisory

**Done when:** opening a PR runs the full quality suite and red means red.
Going forward, no code lands without passing it.

---

## Phase 3 — Domain modeling

Design the data model.

- Sketch every entity, its fields, types, and relationships
- Identify which columns are normalized vs. which live in `jsonb`
- Plan indexes — especially on `MatchParticipant` and `ChampionStatsDaily` which carry the analytics load
- Define the patch-anchoring strategy: how every catalogue entity ties to a `Patch`
- Identify natural keys vs. surrogate keys (e.g., `riot_match_id` as a unique constraint)
- Plan migration ordering — what depends on what
- ADR documenting the model and the key decisions (raw blob storage, denormalization choices)

**Done when:** you can draw the full ERD on a whiteboard from memory, and the
ADR explains every non-obvious choice.

---

## Phase 4 — Riot API integration layer

Build the foundation that every later ingestion phase depends on.

- Riot API client wrapper over Symfony HttpClient
- Region routing abstraction (platform vs. regional routes)
- Rate limiter integration: per-method and per-region app limiters, Redis-backed, shared across workers
- Retry strategy with `Retry-After` header handling
- Structured logging of every outbound call
- Error taxonomy: distinguish transient errors (retry) from permanent (give up) from rate-limit (back off)
- Initial integration test using a dev API key, hitting a low-risk endpoint
- Decide on the local development story: real API calls, recorded fixtures, or a mocked layer

**Done when:** you can make Riot calls from a console command, see them in
logs with timing, and observe the rate limiter throttling correctly under burst.

---

## Phase 5 — Data Dragon synchronization

The catalogue must exist before any match ingestion is meaningful. Match data
references champion and item IDs that need a Patch context.

- Console command to sync Data Dragon for the current version
- Idempotent upsert logic — running twice does nothing harmful
- Version detection: fetch `versions.json`, compare to local state, decide whether to ingest
- Bootstrap mode for backfilling N previous patches
- Cache invalidation when a new patch lands
- Verify image URLs work end-to-end (template helper resolves Champion → ddragon CDN URL)

**Done when:** the database has the current patch's catalogue, the next patch
will sync automatically when Riot releases it, and a Twig template can display
a champion portrait pulled from Riot's CDN.

---

## Phase 6 — Match ingestion pipeline

The core engineering showcase of the project. This is where most of the "real"
work lives.

- Messenger transport configuration with three separate streams
- Failed message handling routed to Doctrine for durability
- Seeding command: fetch leaderboards, dispatch history-fetch messages
- Match-history handler with cooldown checks via Redis dedup layer
- Match-detail handler with transactional persistence and raw payload storage
- Symfony Scheduler wiring for the seed-every-6h cadence
- Cleanup command for bounded raw storage
- Observability: health endpoint exposing queue depth and last-success timestamps per region
- Stress test: run the pipeline for 24-48h, confirm corpus grows, rate limits respected, no duplicates

**Done when:** the pipeline runs unattended for two days, accumulates a
representative match corpus for the current patch, and the metrics endpoint
tells a coherent story about what happened.

---

## Phase 7 — Aggregation layer

Transform the raw corpus into something the UI can read in milliseconds.

- Aggregation command computing `ChampionStatsDaily` from `MatchParticipant`
- DBAL raw SQL with the right groupings, window functions, statistical filters (minimum sample size, etc.)
- Distributed lock ensuring single-leader execution across worker pods
- Scheduler wiring for nightly runs
- Cache invalidation on completion
- Verify aggregation correctness against hand-computed sanity checks
- Decide what statistical guards matter: minimum sample size thresholds, tier bucketing, role assignment heuristics

**Done when:** running the aggregation command populates `ChampionStatsDaily`
with believable numbers, and a manual SQL query against that table returns a
sensible tier list.

---

## Phase 8 — Pillar A: Champion & meta analytics UI

The first user-facing feature. Now the project becomes demonstrable.

- Design system foundations in Tailwind: type scale, color palette, component primitives
- Layout shell: navigation, footer, responsive baseline
- Tier list page with Live Component filter bar (patch, role, region, rank bracket)
- Sortable columns via Stimulus
- Champion detail page with synergies, counters, trend lines
- Patch-diff view comparing two patches
- Loading states, empty states, error states — don't skip these
- Accessibility pass: keyboard navigation, semantic markup, contrast

**Done when:** a non-technical visitor can land on the homepage, navigate to a
tier list, filter it, and click through to a champion detail page that loads
in under 200ms.

---

## Phase 9 — Pillar B: Build optimizer

The analytically meatiest feature.

- Build computation command: frequency analysis of item paths, runes, skill orders
- Confidence-aware filtering: minimum sample sizes, winrate-vs-baseline deltas
- Storage schema for build recommendations (champion × role × patch × optional opponent)
- Build optimizer page with the "against" selector as a Live Component
- Visual presentation: starter items, core path, situational items, runes, skill order
- Confidence indicators visible on every recommendation
- Cache strategy for build queries

**Done when:** picking a champion shows a build that matches what experienced
players actually do, and picking an opposing champion shifts the recommendation
in a way you can defend statistically.

---

## Phase 10 — Production readiness

Before the infrastructure phase, make sure the application is actually
deployable.

- Three runtime modes verified: web, worker, scheduler
- Production Dockerfile with multi-stage build, non-root user, opcache configuration
- Health and readiness endpoints distinct from each other (web liveness ≠ worker queue health)
- Logging in structured JSON to stdout
- Configuration via environment variables, no secrets baked into images
- Database migration strategy for deployments
- Asset compilation and serving in production mode
- Verify the image runs in all three modes from the same Dockerfile

**Done when:** the image builds in CI, runs locally in production mode, and
you can switch between web/worker/scheduler by changing only the command.

---

## Phase 11 — Polish and documentation

The often-skipped phase that separates a portfolio project from a demo.

- README rewritten for someone seeing the project for the first time
- Architecture diagrams in the docs folder, rendered in the README
- ADRs covering every non-obvious decision
- Screenshots or short video of the working application
- Performance notes: what the cache hit rate looks like, query timings, queue throughput
- "Things I'd do differently with more time" section — honesty is a portfolio asset
- Demo data fixture or seed script so reviewers can run it locally without burning their own Riot key

**Done when:** a recruiter can land on the repo, understand what it does in
60 seconds, and have a working local instance running in 5 minutes.

---

## What's deliberately not in this roadmap

- DevOps phases (RKE2, Helm, GitOps, observability stack) — separate roadmap once dev is far enough along
- Pillar C, API Platform, authentication, OAuth — explicit future extensions
- Any feature that doesn't directly serve the two pillars

---

## Rough sequencing

Phases 0–2 are setup and parallel-shippable in a week. Phases 3–7 are the
engineering core and the longest stretch — probably 4–6 weeks at side-project
pace. Phases 8–9 are where it becomes presentable. Phases 10–11 turn it into
a portfolio piece.

Resist the temptation to start Phase 8 before Phase 7 is done. The UI is
satisfying to build but worthless without the data pipeline behind it, and
the pipeline is the harder, more impressive part to show.