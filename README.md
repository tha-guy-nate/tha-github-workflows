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

### `python-publish.yml`

Builds the package, publishes to TestPyPI then PyPI (gated by `testpypi`/`pypi` environments), creates a GitHub Release from the matching `CHANGELOG.md` section, and notifies `tha-wright-stuff` via `repository_dispatch` so its dep-floor bump can pick up the new version. The package name is read from `pyproject.toml`, so no per-repo configuration is needed — the `notify-wright-stuff` job is automatically skipped when the package is `tha-wright-stuff` itself.

#### Usage

In `.github/workflows/publish.yml`, triggered by the tag `python-auto-tag.yml` pushes:

```yaml
name: Publish
on:
  push:
    tags:
      - "v*"

jobs:
  publish:
    permissions:
      id-token: write
      contents: write
    uses: tha-guy-nate/tha-github-workflows/.github/workflows/python-publish.yml@main
    secrets: inherit
```

#### Secrets

| Secret | Required | Description |
|---|---|---|
| `RELEASE_TOKEN` | only if not `tha-wright-stuff` | PAT with `Contents: write`, used to dispatch the `tha-lib-published` event to `tha-wright-stuff` |

#### Environments

Each calling repo must have `testpypi` and `pypi` GitHub environments configured with the reviewer gate (see the family's PyPI publishing conventions).

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
