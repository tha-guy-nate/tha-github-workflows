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
