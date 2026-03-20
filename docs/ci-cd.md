# CI/CD for Python Projects

A practical guide to automating testing, linting, type-checking, and publishing for Python projects using GitHub Actions.

---

## Table of Contents

1. [Why CI/CD?](#1-why-cicd)
2. [Project Prerequisites](#2-project-prerequisites)
3. [Core CI Workflow — Test, Lint & Type-Check](#3-core-ci-workflow--test-lint--type-check)
4. [Caching `uv` and Dependencies](#4-caching-uv-and-dependencies)
5. [Testing Across Python Versions](#5-testing-across-python-versions)
6. [Code Coverage Reporting](#6-code-coverage-reporting)
7. [Publishing to PyPI](#7-publishing-to-pypi)
8. [Security Scanning](#8-security-scanning)
9. [Worked Example — Complete Workflow](#9-worked-example--complete-workflow)
10. [Common Anti-Patterns](#10-common-anti-patterns)

---

## 1. Why CI/CD?

**Continuous Integration (CI)** means every push to your repository automatically runs your test suite, linter, and type-checker. Problems are caught within minutes, before they reach other developers or production.

**Continuous Delivery/Deployment (CD)** means a passing build on a release branch can automatically publish a new package version to PyPI (or deploy a service), with no manual steps.

Without CI/CD, the common failure modes are:

- "Works on my machine" — a missing dependency or a platform-specific bug slips through because only one person ran the tests locally, in their own environment.
- Formatting drift — reviewers spend time on style nits that a linter would catch instantly.
- Broken releases — someone manually runs `pip install .` in the wrong virtual environment and uploads a broken wheel.
- Stale type annotations — `pyright` was never wired up to CI, so incorrect types accumulate undetected.

A well-configured CI pipeline prevents all of these for free, on every push.

---

## 2. Project Prerequisites

The workflows in this guide assume your project follows the conventions established in [Python Common Guidelines](../python/best-practices.md):

- **`pyproject.toml`** as the single source of truth for project metadata and tool config
- **`uv`** for dependency management (with a committed `uv.lock`)
- **`ruff`** for formatting and linting
- **`pyright`** for static type-checking
- **`pytest`** with `pytest-cov` for testing

If your project uses different tooling, adapt the `run:` commands accordingly — the workflow structure stays the same.

Your repository layout should look roughly like this:

```
my-project/
├── src/
│   └── my_package/
├── tests/
├── pyproject.toml
├── uv.lock
└── .github/
    └── workflows/
        ├── ci.yml       # runs on every push / PR
        └── publish.yml  # runs on release tags
```

---

## 3. Core CI Workflow — Test, Lint & Type-Check

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  ci:
    name: Test, Lint & Type-Check
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version-file: pyproject.toml   # reads requires-python

      - name: Install dependencies
        run: uv sync --all-extras --frozen

      - name: Lint & format check (ruff)
        run: |
          uv run ruff check .
          uv run ruff format --check .

      - name: Type-check (pyright)
        run: uv run pyright

      - name: Run tests
        run: uv run pytest tests/ -v
```

Key points:

- `astral-sh/setup-uv` installs `uv` from the official action — no manual `curl | sh` required.
- `--frozen` tells `uv sync` to fail if `uv.lock` is out of date with `pyproject.toml`, catching cases where a developer forgot to commit an updated lockfile.
- `ruff format --check` exits non-zero if any file would be reformatted, enforcing consistent style without modifying files in CI.
- Separating lint, type-check, and tests into named steps (rather than one big `run:` block) means GitHub shows exactly which step failed.

---

## 4. Caching `uv` and Dependencies

By default, every CI run re-downloads and reinstalls all dependencies from scratch. Caching dramatically reduces workflow run times — from ~60 seconds to ~5 seconds for dependency installation on a warm cache.

`astral-sh/setup-uv` supports caching via a single option:

```yaml
      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true
```

This caches the `uv` download cache (typically `~/.cache/uv`) keyed on the contents of `uv.lock`. When the lockfile changes (i.e. dependencies are updated), the cache is automatically invalidated and rebuilt.

> **Tip:** The cache key is based on `uv.lock`, so you get a cache hit on every run that uses the same dependency versions. This is another reason to commit `uv.lock` to version control.

---

## 5. Testing Across Python Versions

Libraries should be tested against every Python version they claim to support. Use a matrix strategy:

```yaml
jobs:
  ci:
    name: Test (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false      # run all versions even if one fails
      matrix:
        python-version: ["3.11", "3.12", "3.13"]

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: uv sync --all-extras --frozen

      - name: Run tests
        run: uv run pytest tests/ -v
```

`fail-fast: false` is important: without it, GitHub cancels the remaining matrix jobs as soon as any one of them fails. Keeping all jobs running lets you see the full picture — maybe Python 3.11 passes but 3.12 doesn't, or vice versa.

> **For applications** (not libraries), testing against a single pinned Python version is usually sufficient. Reserve matrix testing for libraries and packages distributed via PyPI.

---

## 6. Code Coverage Reporting

Track test coverage over time and optionally fail the build if it drops below a threshold.

### Generate a coverage report locally

```bash
uv run pytest tests/ --cov=src/my_package --cov-report=term-missing --cov-report=xml
```

This writes a `coverage.xml` file that can be uploaded to a coverage service.

### Upload to Codecov

[Codecov](https://codecov.io/) is a free coverage tracking service for open-source projects. Add it to your workflow:

```yaml
      - name: Run tests with coverage
        run: |
          uv run pytest tests/ \
            --cov=src/my_package \
            --cov-report=xml \
            --cov-report=term-missing

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          fail_ci_if_error: false   # don't fail the build on upload errors
```

### Enforce a minimum coverage threshold

Add a `[tool.pytest.ini_options]` section in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = "--cov=src/my_package --cov-fail-under=80"
```

With `--cov-fail-under=80`, `pytest` exits non-zero if overall coverage falls below 80 %, causing the CI step to fail.

> **Note:** Coverage thresholds are a useful safety net, but chasing 100% coverage leads to shallow tests written to satisfy the metric rather than to verify behaviour. Aim for meaningful tests over raw coverage numbers. See the [Testing Guide](testing.md) for more on this.

---

## 7. Publishing to PyPI

Use [Trusted Publishing](https://docs.pypi.org/trusted-publishers/) — PyPI's OIDC-based mechanism — instead of storing a long-lived API token as a GitHub secret. With Trusted Publishing, PyPI issues a short-lived token automatically during the workflow run; no secrets to rotate, no risk of token leakage.

### One-time setup on PyPI

1. Go to your project on [pypi.org](https://pypi.org) → **Manage** → **Publishing**.
2. Add a **GitHub Actions** trusted publisher with:
   - **Owner**: your GitHub org/username
   - **Repository**: your repo name
   - **Workflow filename**: `publish.yml`
   - **Environment name**: `pypi` (optional but recommended)

### Publish workflow

Create `.github/workflows/publish.yml`:

```yaml
name: Publish to PyPI

on:
  push:
    tags:
      - "v*"     # triggers on tags like v1.2.3

jobs:
  build:
    name: Build distribution
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version-file: pyproject.toml

      - name: Install dependencies
        run: uv sync --frozen

      - name: Build package
        run: uv build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  publish:
    name: Publish to PyPI
    needs: build
    runs-on: ubuntu-latest
    environment: pypi      # matches the environment name set on PyPI

    permissions:
      id-token: write      # required for OIDC trusted publishing

    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
```

Splitting `build` and `publish` into separate jobs means the built wheel is inspectable as an artifact before it's published — useful for debugging packaging issues.

### Versioning

Tag releases using [semantic versioning](https://semver.org/): `v<MAJOR>.<MINOR>.<PATCH>`.

```bash
git tag v1.2.3
git push origin v1.2.3
```

If you use [hatchling](https://hatch.pypa.io/) as your build backend, you can drive the version from git tags automatically:

```toml
[tool.hatch.version]
source = "vcs"      # reads version from the latest git tag
```

---

## 8. Security Scanning

### Dependency vulnerability scanning

Run `pip-audit` in CI to catch known vulnerabilities in your dependency tree:

```yaml
      - name: Audit dependencies
        run: |
          uv tool install pip-audit
          uv run pip-audit
```

Consider running this as a separate, scheduled workflow (e.g. nightly) so you're alerted to new CVEs even when no code changes are being made:

```yaml
on:
  schedule:
    - cron: "0 6 * * *"   # 06:00 UTC every day
  push:
    branches: [main]
```

### Secret scanning

GitHub's built-in [secret scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) detects accidentally committed API keys and tokens. Enable it in your repository settings under **Security & analysis** — it's free for public repositories.

For additional coverage, [Gitleaks](https://github.com/gitleaks/gitleaks) can be added as a CI step:

```yaml
      - name: Check for leaked secrets (gitleaks)
        uses: gitleaks/gitleaks-action@v2
```

---

## 9. Worked Example — Complete Workflow

Here is a complete, production-ready `ci.yml` combining all of the above:

```yaml
name: CI

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  ci:
    name: Test, Lint & Type-Check (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: uv sync --all-extras --frozen

      - name: Lint (ruff)
        run: uv run ruff check .

      - name: Format check (ruff)
        run: uv run ruff format --check .

      - name: Type-check (pyright)
        run: uv run pyright

      - name: Run tests with coverage
        run: |
          uv run pytest tests/ \
            --cov=src/my_package \
            --cov-report=xml \
            --cov-report=term-missing

      - name: Upload coverage to Codecov
        if: matrix.python-version == '3.12'   # upload once, not per matrix leg
        uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
          fail_ci_if_error: false

  audit:
    name: Dependency Audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5

      - name: Audit dependencies
        run: |
          uv tool install pip-audit
          uv run pip-audit
```

And the matching `publish.yml`:

```yaml
name: Publish to PyPI

on:
  push:
    tags:
      - "v*"

jobs:
  build:
    name: Build distribution
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install uv
        uses: astral-sh/setup-uv@v5
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version-file: pyproject.toml
      - name: Install dependencies
        run: uv sync --frozen
      - name: Build
        run: uv build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  publish:
    name: Publish to PyPI
    needs: build
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
```

---

## 10. Common Anti-Patterns

### Storing PyPI tokens as long-lived secrets

```yaml
# Avoid — token can leak from logs, forks, or compromised runners
- name: Publish
  env:
    TWINE_PASSWORD: ${{ secrets.PYPI_TOKEN }}
  run: twine upload dist/*
```

Use **Trusted Publishing** instead (see [§7](#7-publishing-to-pypi)). It issues a short-lived OIDC token that is scoped to a single workflow run and cannot be replayed.

### Running CI only on `main`

```yaml
# Avoid — problems are caught only after merging
on:
  push:
    branches: [main]
```

Always include `pull_request` triggers so CI runs on every proposed change before it merges:

```yaml
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]
```

### Installing dependencies without `--frozen`

```yaml
# Risky — silently upgrades deps if uv.lock is stale
run: uv sync --all-extras
```

Use `--frozen` in CI. A stale lockfile should be a build failure, not a silent upgrade:

```yaml
run: uv sync --all-extras --frozen
```

### One giant `run:` step

```yaml
# Hard to read; GitHub can't tell you which part failed
- name: Check everything
  run: |
    uv run ruff check .
    uv run ruff format --check .
    uv run pyright
    uv run pytest tests/
```

Split into named steps — GitHub highlights exactly which step failed in the Actions UI:

```yaml
- name: Lint (ruff)
  run: uv run ruff check .
- name: Format check (ruff)
  run: uv run ruff format --check .
- name: Type-check (pyright)
  run: uv run pyright
- name: Run tests
  run: uv run pytest tests/
```

### Skipping CI on documentation-only changes

It can be tempting to skip CI for commits that only change Markdown files. Resist this for anything other than pure content repos — a documentation PR can still break a link checker, a spell checker, or a `mkdocs build` step if you have one wired up.

---

## Further Reading

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [astral-sh/setup-uv action](https://github.com/astral-sh/setup-uv)
- [PyPI Trusted Publishing guide](https://docs.pypi.org/trusted-publishers/)
- [Codecov GitHub Action](https://github.com/codecov/codecov-action)
- [pip-audit](https://github.com/pypa/pip-audit)
- [Semantic Versioning](https://semver.org/)
