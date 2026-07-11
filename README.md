# ci-workflows

Shared reusable GitHub Actions workflows and composite actions for the Defguard repositories.

## Ref policy: pinned semver + Renovate

**Product repos pin a specific semver tag.** No floating `@v1` in committed caller workflow files.

```yaml
# Correct: pinned to a specific release
uses: defguard/ci-workflows/.github/workflows/rehearsal.yml@v1.0.0
```

```yaml
# Incorrect: floating major tag -- do not commit this
uses: defguard/ci-workflows/.github/workflows/rehearsal.yml@v1
```

**Renovate keeps the pin current.** A regex manager in the shared Renovate preset (`renovate/default.json`) scans product-repo workflow files for `DefGuard/ci-workflows/.+@vX.Y.Z` and opens a single grouped PR when a new tag is published. The fix-still-lands-once property is preserved - just behind a merge button.

**`@v1` exists as a convenience pointer** for local/dispatch use, but product repos **must not reference it** in committed workflow files.

### How to cut a release

After a reviewed PR merges to `main`:

```sh
git checkout main
git pull
git tag -a v1.0.1 -m "v1.0.1: description of change"
git tag -f v1
git push origin v1.0.1
git push origin v1 --force
```

Renovate will then open a PR in each product repo to bump the pinned ref from `@v1.0.0` to `@v1.0.1`.

Breaking changes require a new major tag (`v2`) and a migration guide before the old major line is retired.

## Thin-caller pattern

**A product repo's caller workflow is a pure pass-through.** It contains only the `uses:` call plus triggers and concurrency controls - no pre/post jobs, no additional steps.

```yaml
# Example: defguard/.github/workflows/rehearsal.yml
name: Weekly Release Rehearsal
on:
  schedule:
    - cron: '0 8 * * 1'  # Monday 08:00 UTC
  workflow_dispatch:

concurrency:
  group: rehearsal-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rehearsal:
    uses: defguard/ci-workflows/.github/workflows/rehearsal.yml@v1.0.0
    with:
      publish: false
      branch: dev
    secrets:
      MATRIX_WEBHOOK_URL: ${{ secrets.MATRIX_WEBHOOK_URL }}
      GHCR_READ_TOKEN: ${{ secrets.GHCR_READ_TOKEN }}
```

**Rationale:** this guarantees rehearsal and release call the exact same underlying jobs with no hidden divergence. If a repo needs a repo-specific step, extract it into ci-workflows behind a parameter - do not add it to the caller.

### Composite actions

Composite actions are called from steps within any workflow (reusable or caller):

```yaml
steps:
  - uses: defguard/ci-workflows/actions/alert@v1.0.0
    with:
      alert-key: rehearsal-dev
      title: "Rehearsal failed on dev"
      labels: release-blocker
      github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Parity rule

**Any change to a release build path must land in this repo first.**

A PR touching a product repo's release caller workflow (beyond a ref bump) is a red flag - the shared workflow here must be updated instead. The weekly release rehearsal exercises the shared workflow automatically. Rehearsal and release must call the same underlying jobs, so divergence between what is rehearsed and what actually ships is impossible.

## Repo layout

```
ci-workflows/
  .github/
    workflows/       # Reusable workflow_call workflows
  actions/           # Composite actions
  renovate/          # Shared Renovate preset (default.json)
  README.md
```

## Contents

| Path | Type | Purpose |
|------|------|---------|
| `.github/workflows/lint.yml` | CI workflow | actionlint + shellcheck on every PR to this repo |
| `.github/workflows/hello-world.yml` | Reusable workflow | Minimal `workflow_call` probe for cross-repo caller validation |
| `actions/` | Composite actions | Shared actions for alerting, readiness checks, etc. (populated by subsequent tickets) |
| `renovate/default.json` | Renovate preset | Shared config extended by all product repos |

## Developing

Every PR to this repo runs:

- **actionlint** - static analysis of all workflow YAML files
- **shellcheck** - linting of all shell scripts in composite actions

New workflows and actions are tested via manual dispatch before their schedule or trigger goes live.
