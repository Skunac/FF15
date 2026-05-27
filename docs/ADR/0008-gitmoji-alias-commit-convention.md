# ADR 0008: Gitmoji alias commit convention

**Status:** Accepted
**Date:** 2026-05-27

## Context

FF15 needs a single, enforceable commit message convention before the volume
of commits makes any retrofit painful. Two pressures motivate this now:

- The existing history mixes styles — some commits use literal Unicode emoji
  (`💚 add codespell configuration`), others use gitmoji aliases
  (`:memo: add architecture decision records`). A reader skimming `git log`
  cannot easily classify commits, and grepping for a category requires
  knowing every variant in use.
- Dependabot is currently configured with `commit-message.prefix: "chore"`
  (Conventional Commits style), which clashes with the rest of the history.

A choice must be made between the two dominant conventions — Conventional
Commits and Gitmoji — and enforcement must follow, because conventions that
aren't enforced decay within a few months on a solo project.

## Decision

Commit message subjects must start with one or more gitmoji aliases (e.g.,
`:sparkles:`, `:bug:`, `:memo:`), followed by a space and a short
description:

```
:sparkles: add tier list page
:bug: fix off-by-one in MatchParticipant aggregation
:arrow_up: bump symfony/framework-bundle from 8.0 to 8.1
```

The full alias list lives at [gitmoji.dev](https://gitmoji.dev).

Enforcement is two-layered:

- **Local:** a POSIX-shell `commit-msg` hook in [.githooks/](../../.githooks/),
  opted into per clone with `git config core.hooksPath .githooks`.
- **CI:** a [`commit-lint`](../../.github/workflows/commit-lint.yaml) workflow
  that validates every commit in a PR (and on push to `main`) using the same
  regex as the local hook. The CI job is the authoritative gate; the local
  hook is for fast feedback.

[.github/dependabot.yaml](../../.github/dependabot.yaml) is updated so that
its commit messages start with `:arrow_up:`, matching the convention without
requiring an allow-list.

Merge, revert, fixup, squash, and amend autosquash commits are allow-listed
(they have machine-generated subjects and would otherwise need every
contributor to manually rewrite them).

## Consequences

**Positive:**

- A glance at `git log` immediately reveals the nature of each commit by
  shape — `:sparkles:` for features, `:bug:` for fixes, `:arrow_up:` for
  dependencies — without needing to read past the emoji.
- The convention costs nothing to follow once familiar; the aliases are
  short and the standard list is small.
- No new language toolchain. The hook is POSIX shell; the CI step is a
  shell loop. The PHP-only repository stays PHP-only.
- Visual signal on a portfolio repo. A clean, consistent commit log reads
  as care.

**Negative:**

- Less machine-parseable than Conventional Commits. Changelog tools such as
  `release-please` or `semantic-release` expect `feat:` / `fix:` prefixes
  and would need a custom mapper to consume `:sparkles:` / `:bug:`. If
  release automation is added later, this becomes work.
- Gitmoji aliases require the reader to know what each emoji means. A
  developer unfamiliar with the convention needs the cheat sheet open until
  the common ones (`:sparkles:`, `:bug:`, `:memo:`, `:arrow_up:`,
  `:recycle:`, `:lipstick:`, `:lock:`) are memorized.
- The allow-list for merge/revert/fixup commits is a small surface for
  drift; if a future GitHub feature introduces another auto-generated
  subject format, the hook will reject it until updated.

## Rejected alternatives

**Conventional Commits (`feat:`, `fix:`, `chore:`).** Rejected on personal
preference and visual ergonomics. Conventional Commits is the more standard
choice and integrates effortlessly with changelog tooling, but the
maintainer finds the prefixes visually heavier than gitmoji aliases at
glance-scanning scale. The release-automation argument is real but does not
apply until the project ships releases — none are planned in the current
roadmap.

**Hybrid (`:sparkles: feat: add foo`).** Rejected as verbose. Combining
both conventions buys the machine-parseability of Conventional Commits and
the visual scan of gitmoji, but doubles the prefix length on every commit
and asks the writer to make two categorization decisions where one would
do.

**No enforcement, only documentation.** Rejected based on history: the
project is six commits old and already shows convention drift. Documented-
but-not-enforced conventions decay; the CI gate exists precisely so that
the decision survives the maintainer's discipline drifting at 11pm.

## Future review

Re-evaluate if any of the following becomes true:

- Release automation is added (e.g., a changelog generated from commit
  history). At that point, weigh the cost of a custom gitmoji-to-changelog
  mapper against migrating to Conventional Commits.
- The project gains regular contributors who push back on the convention.
  Gitmoji is a strong personal preference; in a team, the convention should
  be the team's, not the maintainer's.
- The allow-list grows beyond merge/revert/fixup/squash/amend. That would
  indicate the regex is too strict for real-world usage and needs a rethink.
