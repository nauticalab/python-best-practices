# Docker for Python Development

A concise guide to using Docker effectively in Python projects — from writing lean Dockerfiles to local development with Compose.

---

## Table of Contents

1. [Why Use Docker for Python Development](#1-why-use-docker-for-python-development)
2. [Writing a Good Dockerfile](#2-writing-a-good-dockerfile)
3. [Managing Dependencies](#3-managing-dependencies)
4. [Docker Compose for Local Development](#4-docker-compose-for-local-development)
5. [Common Pitfalls](#5-common-pitfalls)
6. [Production-Ready Dockerfile Example](#6-production-ready-dockerfile-example)

---

## 1. Why Use Docker for Python Development

Python's flexibility is also a source of friction: different machines carry different Python versions, conflicting system packages, and subtly different environments. Docker solves this by packaging your application with everything it needs to run — interpreter, dependencies, config — into a portable image.

**Reproducibility.** "Works on my machine" stops being an excuse. Every developer, CI runner, and production host boots the exact same filesystem snapshot.

**Isolation.** Each project runs in its own container. No more fighting over conflicting package versions between projects, and no risk of one project's `pip install` polluting another's environment.

**CI/CD parity.** Your test suite runs inside the same image that ships to production. If it passes in CI, it will behave the same way when deployed — no more "but the CI environment is different" surprises.

---

## 2. Writing a Good Dockerfile

### Choose a slim base image

The official `python` images come in several flavours. Prefer `python:<version>-slim` for most use cases — it is a stripped-down Debian image that is roughly 50 MB vs ~350 MB for the full image.

```dockerfile
# Good — lean runtime base
FROM python:3.12-slim

# Avoid unless you specifically need OS-level build tools
# FROM python:3.12
```

> **Tip:** Pin the minor version (`3.12-slim`, not `3-slim`) so a base-image update never silently changes your Python version.

### Use multi-stage builds

Multi-stage builds let you compile or install build-time tools in one stage and copy only the final artefacts into a clean runtime stage. The result is a smaller, more secure image — build tools like `gcc`, `git`, or `pip-tools` never end up in production.

```dockerfile
# ── Stage 1: build ────────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

# Install uv for fast dependency installation
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# ── Stage 2: runtime ──────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only the installed virtualenv from the builder
COPY --from=builder /build/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"

COPY src/ ./src/

CMD ["python", "-m", "myapp"]
```

### Leverage layer caching

Docker rebuilds a layer only when its inputs change. Copy dependency files **before** application source code so that a code-only change does not re-install all packages.

```dockerfile
# Good — dependencies are cached unless pyproject.toml / uv.lock changes
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY src/ ./src/

# Bad — any code change invalidates the package install layer
# COPY . .
# RUN uv sync --frozen --no-dev
```

### Set Python environment variables

Two environment variables should almost always be set in a Python Dockerfile:

```dockerfile
# Prevent Python from writing .pyc files (no benefit in a container)
ENV PYTHONDONTWRITEBYTECODE=1

# Ensure stdout/stderr are unbuffered so logs appear immediately
ENV PYTHONUNBUFFERED=1
```

---

## 3. Managing Dependencies

### Choosing a dependency tool

| Tool | Use case | Lockfile |
|------|----------|----------|
| `uv` | Recommended for all new projects | `uv.lock` |
| `pip-tools` | Legacy projects or teams already using it | `requirements.txt` (compiled) |
| `pip freeze` | Quick scripts only | None |

**`uv` (recommended).** Generate a `uv.lock` file by running `uv sync`. In your Dockerfile, pass `--frozen` to ensure the exact locked versions are installed:

```dockerfile
RUN uv sync --frozen --no-dev
```

**`pip-tools`.** Compile a pinned `requirements.txt` from `requirements.in` and commit both files:

```bash
pip-compile requirements.in -o requirements.txt
```

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

**Avoid bare `pip freeze`** for anything beyond a throwaway script — it captures the entire environment, including transitive dev tools, and provides no dependency resolution.

### Copy requirements before source code

Regardless of the tool you use, always copy dependency files before copying application code. This keeps the expensive "install packages" layer cached across most builds:

```dockerfile
# 1. Dependencies (cached unless lock file changes)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# 2. Application code (invalidates only the layers below this line)
COPY src/ ./src/
```

---

## 4. Docker Compose for Local Development

Docker Compose is the standard tool for running multi-container local environments (app + database + cache, etc.). For development, the key addition over a plain `docker run` is **volume mounts** — your local source tree is mounted into the container so code changes take effect without rebuilding the image.

### Basic `compose.yaml`

```yaml
services:
  app:
    build: .
    volumes:
      # Mount source code for hot-reload
      - ./src:/app/src
    ports:
      - "8000:8000"
    environment:
      - LOG_LEVEL=DEBUG
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Development overrides

Keep production-safe defaults in `compose.yaml` and layer development-specific settings (hot reload, debug ports, relaxed timeouts) in a `compose.override.yaml` file. Docker Compose merges the two automatically when you run `docker compose up`:

```yaml
# compose.override.yaml  (not committed, or committed with dev-only values)
services:
  app:
    command: ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--reload"]
    environment:
      - DEBUG=true
```

> **Tip:** Add `compose.override.yaml` to `.gitignore` if it contains local paths or secrets. Provide a `compose.override.yaml.example` that teammates can copy.

---

## 5. Common Pitfalls

### Running as root

By default, processes in a Docker container run as root. If your container is compromised, an attacker gains root-level access to the container filesystem (and potentially the host, if volumes are mounted). Always create and switch to a non-root user:

```dockerfile
RUN useradd --create-home --shell /bin/bash appuser
USER appuser
```

### Bloated images

Common causes of unnecessarily large images:

- Using `python:3.12` instead of `python:3.12-slim`
- Leaving build tools (`gcc`, `git`, `curl`) in the final stage — use multi-stage builds
- Copying the entire repository with `COPY . .` instead of only what is needed
- Not cleaning up apt caches after installing OS packages:

```dockerfile
# Always clean up in the same RUN layer to avoid storing the cache in a layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```

### Missing `.dockerignore`

Without a `.dockerignore`, `COPY . .` sends your entire working directory to the Docker daemon — including `.git/`, `__pycache__/`, `.venv/`, test fixtures, and secrets. Create a `.dockerignore` alongside your `Dockerfile`:

```
.git/
.venv/
__pycache__/
*.pyc
*.pyo
.pytest_cache/
.ruff_cache/
.mypy_cache/
htmlcov/
dist/
build/
.env
.env.*
compose.override.yaml
```

### Ignoring `PYTHONUNBUFFERED`

Without `ENV PYTHONUNBUFFERED=1`, Python buffers stdout. In a container, this means log output may not appear until the buffer flushes — or at all if the container crashes. Always set this variable.

---

## 6. Production-Ready Dockerfile Example

A complete example using multi-stage builds, `uv`, a non-root user, and sensible environment defaults.

```dockerfile
# .dockerignore should exist alongside this file — see Section 5.

# ── Stage 1: install dependencies ────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Bring in uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install dependencies into a virtualenv (no dev deps, no editable installs)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# ── Stage 2: runtime image ────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH"

# Non-root user
RUN useradd --create-home --shell /bin/bash appuser

# Copy virtualenv from builder
COPY --from=builder /build/.venv /app/.venv

# Copy application source
COPY src/ ./src/

# Switch to non-root user before the final CMD
USER appuser

EXPOSE 8000

CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Further Reading

- [Docker Documentation — Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Documentation — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [uv — Docker integration guide](https://docs.astral.sh/uv/guides/integration/docker/)
- [Docker Compose — Getting started](https://docs.docker.com/compose/gettingstarted/)
