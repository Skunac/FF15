# Git hooks

Project-managed git hooks. Enable once per clone:

```sh
make hooks
```

(Equivalent to `git config core.hooksPath "$(git rev-parse --show-toplevel)/.githooks"`.)

## Hooks

- **`commit-msg`** — rejects commit messages that don't follow the
  gitmoji-alias convention. See
  [docs/ADR/0008](../docs/ADR/0008-gitmoji-alias-commit-convention.md) for
  the rationale, or [gitmoji.dev](https://gitmoji.dev) for the alias list.

The same validation runs server-side via
[.github/workflows/commit-lint.yaml](../.github/workflows/commit-lint.yaml),
so the local hook is for fast feedback — it is not the only enforcement.
