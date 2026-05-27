# ADR 0005: No API Platform in v1

**Status:** Accepted
**Date:** 2026-05-27

## Context

API Platform is a popular Symfony bundle for building REST and GraphQL APIs
with auto-generated OpenAPI documentation. It is frequently included in
Symfony projects by default.

FF15 is server-rendered (see ADR-0003) and has no SPA, no mobile client, and
no documented external API consumer. The architecture's read path is:
controller → repository → Twig render.

A choice must be made about whether to introduce API Platform in v1.

## Decision

We will not include API Platform in v1.

The architecture will use plain Symfony controllers returning Twig responses
for the UI, and plain Doctrine repositories for data access. No JSON or
GraphQL endpoints will be exposed in v1.

## Consequences

**Positive:**

- One way of doing things — controllers render Twig, repositories return
  entities. No parallel API resource layer to maintain.
- No serialization concerns, no API versioning concerns, no CORS concerns.
- Smaller dependency surface, faster CI, simpler deployment.
- Documentation effort focuses on what the project actually does, not on
  endpoints that have no callers.

**Negative:**

- Adding an API later requires retrofitting. Mitigated by the fact that the
  domain model and repository layer are designed to be reusable; an API
  Platform layer can sit on top of them without restructuring.
- The portfolio doesn't visibly demonstrate API Platform proficiency. Other
  Symfony depth signals (Messenger, Scheduler, UX components, raw DBAL,
  RateLimiter) compensate.

## Rejected alternatives

**Include API Platform in v1 for the portfolio showcase value.** Rejected
because adding a tool to a project just to demonstrate familiarity with it is
the wrong incentive. The project's job is to solve its problems well; tooling
choices should follow the problems. Demonstrating restraint and tool selection
judgment is a more valuable signal than checkbox feature inclusion.

**Include API Platform with internal-only endpoints.** Rejected because the
endpoints would have no real callers and would add maintenance burden without
serving any user need.

## Future review

API Platform earns its place once the project has:

- A documented external consumer (public API, mobile app, partner integration), or
- Internal services that benefit from a versioned, typed contract (none planned).

A future ADR should propose adding API Platform when one of these triggers
materializes. The build-out is listed in
[DEV_ROADMAP.md](../DEV_ROADMAP.md) under future extensions.