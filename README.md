# tha-github-workflows

Shared GitHub Actions reusable workflows for the [`tha-*` Python library family](https://github.com/tha-guy-nate/tha-wright-stuff).

## Workflows

All workflows below are reusable, called via `workflow_call` from each repo's own `.github/workflows/*.yml`.

### `python-ci.yml`

Runs the full CI suite across a matrix of Python versions: lint, format check, tests with coverage, type check, dependency check, vulnerability check, and build.

#### Usage

In `.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/python-ci.yml@main
```

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `python-versions` | string | `["3.10","3.11","3.12","3.13","3.14"]` | JSON array of Python versions to test |

To restrict the matrix:

```yaml
jobs:
  ci:
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/python-ci.yml@main
    with:
      python-versions: '["3.12", "3.13"]'
```

#### Steps

1. `actions/checkout@v7`
2. `astral-sh/setup-uv@v8.2.0` — installs uv
3. `uv python install` — pins the matrix Python version
4. `uv sync --extra dev` — installs the package and dev deps
5. `ruff check` — lint
6. `ruff format --check` — format
7. `pytest --cov=src --cov-report=term-missing --cov-report=xml` — tests + coverage
8. Upload coverage to Codecov (`codecov/codecov-action@v5`, tokenless OIDC) — only on the `3.12` matrix leg, to avoid duplicate uploads
9. `mypy` — type check
10. `deptry` — dependency check; only runs if the repo has a `[tool.deptry]` section in `pyproject.toml`, otherwise skipped
11. `pip-audit` — vulnerability check
12. `uv build` — build distribution

### `python-auto-tag.yml`

Detects a version bump in `pyproject.toml` (comparing HEAD against HEAD~1) and, if the version changed, creates and pushes a `vX.Y.Z` git tag. Skips if the version is unchanged or the tag already exists. Pushing the tag is what triggers each repo's own `publish.yml` (see below).

#### Usage

In `.github/workflows/ci.yml`, as a job that runs after CI passes on `push`:

```yaml
jobs:
  auto-tag:
    if: github.event_name == 'push'
    needs: [ci]
    permissions:
      contents: write
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/python-auto-tag.yml@main
    secrets: inherit
```

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `RELEASE_TOKEN` | yes | PAT with `Contents: write`, used to push the tag as `github-actions[bot]` |

### `pre-commit.yml`

Runs the repo's `.pre-commit-config.yaml` via `pre-commit/action@v3.0.1` on Python 3.12, mirroring the local pre-commit hook in CI.

#### Usage

```yaml
jobs:
  pre-commit:
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/pre-commit.yml@main
```

### Why `publish.yml` is NOT centralized here

This was tried (2026-07-05) and reverted after a real release test (`tha-map-runner` v0.2.12) failed: **PyPI's Trusted Publishing (OIDC) does not support reusable/`workflow_call` workflows.** The OIDC token's `job_workflow_ref` claim points at the reusable workflow's path instead of the calling repo's own `publish.yml`, so PyPI's Trusted Publisher match fails with `invalid-publisher`:

```
The claims in this token suggest that the calling workflow is a reusable workflow... Reusable workflows are
not currently supported by PyPI's Trusted Publishing functionality, and are subject to breakage.
```

See [pypa/gh-action-pypi-publish#166](https://github.com/pypa/gh-action-pypi-publish/issues/166). This is a hard platform limitation, not a config mistake — every repo's `publish.yml` must stay a standalone file with its own `build`/`publish-testpypi`/`publish-pypi`/`create-release`/`notify-wright-stuff` jobs. Don't re-attempt this without checking whether PyPI has added reusable-workflow support first.

### `auto-assign-pr.yml`

On PR open/reopen: assigns the PR author as assignee (skipped for bot authors like `dependabot[bot]`/`github-actions[bot]`), and requests `tha-guy-nate` as a reviewer — except when he's the author himself (GitHub blocks self-review-requests).

#### Usage

In `.github/workflows/auto-assign-pr.yml`:

```yaml
name: Auto-assign PR
on:
  pull_request:
    types: [opened, reopened]

jobs:
  assign:
    permissions:
      pull-requests: write
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/auto-assign-pr.yml@main
```

### `check-yanked-floors.yml`

Scans this repo's `tha-*>=X.Y.Z` (and `tha-*[extra]>=X.Y.Z`) dependency floors against PyPI's per-release `yanked` status. Opens or updates a single tracking issue titled "Yanked dependency floor(s) detected" on the calling repo when a floor points at a yanked release; auto-closes it once resolved.

PyPI has no public API to yank a release programmatically (web UI only, CSRF-protected) — this covers detection and notification, not the yank action itself.

#### Usage

In `.github/workflows/yanked-floor-check.yml`:

```yaml
name: Yanked Floor Check
on:
  schedule:
    - cron: '30 9 * * *'
  workflow_dispatch:
jobs:
  check:
    permissions:
      issues: write
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/check-yanked-floors.yml@main
```
