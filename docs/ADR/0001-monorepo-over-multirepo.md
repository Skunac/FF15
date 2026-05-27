# ADR 0001: Monorepo over multirepo

**Status:** Accepted
**Date:** 2026-05-27

## Context

FF15 consists of one Symfony application (with three runtime modes — web,
worker, scheduler), its Kubernetes infrastructure (Helm charts, cluster
manifests), and project documentation. The project is maintained by one
developer and exists primarily as a portfolio piece showcasing both
development and operations work.

A choice must be made between keeping everything in a single repository or
splitting across multiple repositories (e.g., `ff15-app`, `ff15-infra`,
`ff15-docs`).

## Decision

We will use a single repository, `ff15`, with top-level directories `app/`,
`infra/`, and `docs/`.

Per-component CI isolation will be achieved via GitHub Actions path filters
rather than separate repositories.

## Consequences

**Positive:**

- Cross-cutting changes (e.g., adding a Messenger transport that requires both
  application config and Helm values updates) can be made atomically in a
  single PR.
- A reader landing on the repository immediately sees both the dev and devops
  pillars in the top-level tree — the entire project pitch is visible in one
  `ls`.
- Single source of truth for documentation; no question of where the README
  lives or which repo to clone first.
- No version coordination needed between separate repositories.
- One CI configuration, one set of branch protection rules, one issue tracker.

**Negative:**

- Cannot grant access to one part of the project without exposing all of it.
  Not relevant for a solo portfolio project, but would matter in a team
  context.
- CI configuration is shared, which could become unwieldy if the project grows
  significantly in scope.

## Rejected alternatives

**Separate repositories per concern (`ff15-app`, `ff15-infra`).** Rejected
because the version coordination overhead and the discoverability cost exceed
any isolation benefit at this scale. Multirepo earns its place with multiple
teams, divergent release cadences, or polyglot stacks — none of which apply
here.

**Monorepo with dedicated tooling (Nx, Turborepo, Bazel).** Rejected because
those tools target polyglot monorepos with shared build graphs. FF15 has one
Composer-managed PHP application and some YAML. Adding monorepo tooling on top
would be ceremony without payoff.

## Future review

Re-evaluate if the project gains contributors who need scoped access, or if
the infrastructure layer grows large enough (e.g., shared platform components
used by multiple applications) to warrant its own release cycle. Extraction
via `git filter-repo` remains available if needed.