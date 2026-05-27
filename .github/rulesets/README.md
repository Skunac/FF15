# Rulesets

GitHub branch-protection rulesets, version-controlled so the rules live with
the code that depends on them.

## Files

- **`main.json`** — protects the `main` branch. Strict mode: every change
  arrives via a PR with green CI on a linear history. No bypasses, no force
  pushes, no direct deletions.

## Applying

Rulesets are not applied automatically by GitHub. Apply with the `gh` CLI
after the referenced status checks have run at least once on the repo
(GitHub will not let you require a check it has never seen):

```sh
# First time:
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/rulesets \
  --input .github/rulesets/main.json

# Updating an existing ruleset (find its id first):
gh api /repos/{owner}/{repo}/rulesets
gh api \
  --method PUT \
  -H "Accept: application/vnd.github+json" \
  /repos/{owner}/{repo}/rulesets/{ruleset_id} \
  --input .github/rulesets/main.json
```

## Required status checks

`main.json` requires three checks by name (must match the `name:` of the job
in the workflow file):

- `Tests` — from [.github/workflows/ci.yaml](../workflows/ci.yaml)
- `Lint` — from [.github/workflows/ci.yaml](../workflows/ci.yaml)
- `Commit lint` — from
  [.github/workflows/commit-lint.yaml](../workflows/commit-lint.yaml)

If a check is renamed, update both the workflow `name:` field and the
corresponding entry in `main.json`. GitHub matches required checks by exact
string.
