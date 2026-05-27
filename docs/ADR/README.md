# Architecture Decision Records

This directory contains the Architecture Decision Records (ADRs) for FF15.

An ADR captures a single architectural decision: its context, the choice
made, the alternatives considered, and the consequences. ADRs are
**immutable** once accepted — if a decision changes, a new ADR supersedes
the old one rather than editing it.

The format used here is a lightly extended version of the original
[Michael Nygard template](https://github.com/joelparkerhenderson/architecture-decision-record).

## Index

| ID | Title | Status |
|----|-------|--------|
| [0001](0001-monorepo-over-multirepo.md) | Monorepo over multirepo | Accepted |
| [0002](0002-symfony-8-over-lts.md) | Symfony 8.0 over the current LTS | Accepted |
| [0003](0003-twig-and-live-components-over-spa.md) | Twig + Live Components over a separate SPA | Accepted |
| [0004](0004-redis-five-role-separation.md) | Five-role separation of Redis usage | Accepted |
| [0005](0005-no-api-platform-in-v1.md) | No API Platform in v1 | Accepted |
| [0006](0006-aggregate-on-write-not-on-read.md) | Aggregate on write, not on read | Accepted |
| [0007](0007-dont-rehost-data-dragon-images.md) | Link Data Dragon images from Riot's CDN; do not rehost | Accepted |
| [0008](0008-gitmoji-alias-commit-convention.md) | Gitmoji alias commit convention | Accepted |

## Conventions

- Files are named `NNNN-kebab-case-title.md` where `NNNN` is the
  zero-padded sequence number.
- Numbers are assigned at creation time and never change.
- Status values: `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR-NNNN`.
- An ADR superseded by another should update its status line to point at the
  replacement, but otherwise remain unchanged.

## When to write a new ADR

Write one when:

- A non-obvious choice is being made between real alternatives.
- A choice is being made that someone could reasonably question later.
- A constraint is being deliberately accepted ("we know X is suboptimal, but...").

Do not write one for:

- Routine implementation details (which Doctrine annotation to use, what to
  name a service).
- Decisions that follow trivially from an existing ADR.
- Things that aren't actually decisions ("we used PHP because it's a Symfony
  project").