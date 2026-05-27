# ADR 0002: Symfony 8.0 over the current LTS

**Status:** Accepted
**Date:** 2026-05-27

## Context

Symfony 8.0 is the current stable release as of mid-2026. Symfony 7.4 became
the new LTS in November 2025, with three years of bug fixes and one year of
security support. Symfony 6.4 remains supported for security through November
2027.

FF15 is a portfolio project. Stack choices serve two purposes: shipping the
project itself, and signaling current technical awareness to readers of the
repository.

A choice must be made between targeting the LTS (7.4) for maximum stability
and longevity, or the current stable (8.0) for modernity.

## Decision

We will target Symfony 8.0, which requires PHP 8.4.

## Consequences

**Positive:**

- Access to the latest framework features and ergonomics.
- Demonstrates current technical awareness, which is a non-trivial signal for
  a portfolio project.
- PHP 8.4 features (property hooks, asymmetric visibility, readonly classes)
  are available throughout the codebase.
- Aligns with the project's general principle of choosing modern tools where
  the cost is acceptable.

**Negative:**

- Symfony 8.0's standard support ends in July 2026, after which a migration to
  the next LTS (8.4) will be required. For a portfolio project this is an
  acceptable maintenance burden — Symfony major-to-LTS upgrades within the
  same major are generally straightforward.
- Some third-party bundles may lag in their Symfony 8 compatibility. Bundle
  selection will need to verify support before adoption.
- PHP 8.4 is required on every environment (local, CI, production). Not a
  significant concern given Docker-based deployment.

## Rejected alternatives

**Symfony 7.4 LTS.** Rejected because the stability benefit is marginal for a
portfolio project with no production users, and the modernity signal of using
the current stable is more valuable than the longer support window.

**Symfony 6.4 LTS.** Rejected for the same reason as 7.4, with the additional
downside of being two majors behind.

## Future review

Plan a migration to Symfony 8.4 LTS once it releases (expected November 2026,
based on Symfony's release cadence). Document the migration as a separate ADR
when undertaken.