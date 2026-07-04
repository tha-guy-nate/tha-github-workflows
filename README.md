# tha-github-workflows

Shared GitHub Actions reusable workflows for the [`tha-*` Python library family](https://github.com/tha-guy-nate/tha-wright-stuff).

## Workflows

### `python-ci.yml`

Runs the full CI suite across a matrix of Python versions: lint, format check, type check, tests, and build.

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
7. `pytest` — tests
8. `mypy` — type check
9. `uv build` — build distribution

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
