# ci-workflows

Shared reusable GitHub Actions workflows and composite actions for the Defguard repositories.

## Caller pattern

### Reusable workflows

Product repos reference workflows from this repo by major tag:

```yaml
uses: defguard/ci-workflows/.github/workflows/rehearsal.yml@v1
```

Passing inputs and secrets explicitly per the workflow's contract:

```yaml
jobs:
  rehearsal:
    uses: defguard/ci-workflows/.github/workflows/rehearsal.yml@v1
    with:
      publish: false
    secrets:
      MATRIX_WEBHOOK: ${{ secrets.MATRIX_WEBHOOK }}
```

### Composite actions

```yaml
steps:
  - uses: defguard/ci-workflows/actions/alert@v1
    with:
      alert-key: rehearsal-dev
      title: "Rehearsal failed on dev"
      labels: release-blocker
      github-token: ${{ secrets.GITHUB_TOKEN }}
      matrix-webhook: ${{ secrets.MATRIX_WEBHOOK }}
```

## Versioning

This repo uses a **moving major tag** convention:

- `v1` always points to the latest stable release on the v1 line
- `v1.x.x` tags pin a specific release
- Product repos pin `@v1`; the tag is moved deliberately after review

To advance `v1` to a new commit after a reviewed change:

```sh
git tag -a v1.x.x -m "v1.x.x"
git tag -f v1
git push origin v1.x.x
git push origin v1 --force
```

Breaking changes require a new major tag (`v2`) and a migration guide before the old major tag is retired.

## Parity rule

**Any change to a release build path must land in this repo first.**

A PR touching a product repo's release caller workflow is a red flag - update the shared workflow here instead, so the weekly release rehearsal exercises the change automatically. Rehearsal and release must call the same underlying jobs.

## Contents

| Path | Type | Purpose |
|------|------|---------|
| `.github/workflows/lint.yml` | CI workflow | actionlint + shellcheck on every PR to this repo |
| `.github/workflows/hello-world.yml` | Reusable workflow | Caller-pattern probe (temporary; replaced by real workflows as they are added) |

More workflows and composite actions will be added here as the release-process smoothing initiative progresses.

## Developing

Every PR to this repo runs:

- **actionlint** - static analysis of all workflow YAML files
- **shellcheck** - linting of all shell scripts in composite actions

New workflows and actions are tested via manual dispatch before their schedule or trigger goes live.
