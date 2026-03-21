# Python Common Guidelines

A practical reference for writing clean, idiomatic, and maintainable Python code.

---

## Table of Contents

1. [Code Style & Formatting](#1-code-style--formatting)
2. [Type Hints & Static Analysis](#2-type-hints--static-analysis)
3. [Project Structure & Packaging](#3-project-structure--packaging)
4. [Dependency Management](#4-dependency-management)
5. [Testing](#5-testing)
6. [Documentation](#6-documentation)
7. [Error Handling & Logging](#7-error-handling--logging)
8. [Performance](#8-performance)
9. [Security](#9-security)
10. [Common Anti-Patterns to Avoid](#10-common-anti-patterns-to-avoid)

---

## 1. Code Style & Formatting

### Follow PEP 8
[PEP 8](https://peps.python.org/pep-0008/) is the official Python style guide. Key rules:
- 4 spaces per indentation level (no tabs)
- Maximum line length of **88 characters** (Black default; PEP 8 allows 79)
- Two blank lines between top-level definitions, one between methods
- Imports grouped: stdlib → third-party → local, each group separated by a blank line

### Use an Autoformatter
Enforce consistent style automatically — don't leave it to human judgment.

```bash
# Recommended: Ruff (fast, replaces black + isort + flake8)
uv add --dev ruff
ruff format .      # format files
ruff check .       # lint
```

Configure in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B"]   # pycodestyle, pyflakes, isort, pyupgrade, bugbear
ignore = ["E501"]  # line-too-long handled by formatter

[tool.black]
line-length = 88
```

### Automate Formatting with pre-commit Hooks

[**pre-commit**](https://pre-commit.com/) is a framework for managing Git hooks. Configure it once and it automatically runs formatters, linters, and (optionally) tests every time you `git commit` — catching issues before they ever reach the remote.

```bash
uv add --dev pre-commit
pre-commit install          # installs the hook into .git/hooks/pre-commit
pre-commit run --all-files  # run manually across the whole repo
```

Add a `.pre-commit-config.yaml` at the project root:

```yaml
repos:
  # Ruff: lint + format (replaces black, isort, flake8)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff          # lint and auto-fix
        args: [--fix]
      - id: ruff-format   # format (black-compatible)

  # Pyright: type checking
  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false

  # Optional: run fast unit tests before every commit
  - repo: local
    hooks:
      - id: pytest-fast
        name: pytest (fast subset)
        entry: uv run pytest tests/ -x -q --timeout=10
        language: system
        pass_filenames: false
        stages: [pre-commit]
```

> **Tip:** Keep pre-commit hooks fast (< 10 s). Run full test suites in CI, not on every local commit. Use `stages: [pre-push]` if you want heavier checks only on push.

Commit `.pre-commit-config.yaml` to version control so the whole team uses the same hooks.

### Naming Conventions

| Entity | Convention | Example |
|--------|-----------|---------|
| Variables & functions | `snake_case` | `user_count`, `get_user()` |
| Classes | `PascalCase` | `UserProfile` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_RETRIES` |
| Private members | `_leading_underscore` | `_internal_cache` |
| Dunder/magic methods | `__double_underscore__` | `__init__`, `__repr__` |
| Modules / packages | `short_lowercase` | `utils`, `db_client` |

---

## 2. Type Hints & Static Analysis

### Always Annotate Public APIs
Type hints improve IDE support, catch bugs early, and serve as documentation.

```python
# Good
def fetch_user(user_id: int, *, active_only: bool = True) -> User | None:
    ...

# Avoid — no type information for callers
def fetch_user(user_id, active_only=True):
    ...
```

### Use Modern Type Syntax (Python 3.10+)

```python
# Prefer union syntax over Union/Optional from typing
def process(value: int | str | None) -> list[str]:
    ...

# Use built-in generics (3.9+) instead of typing.List, typing.Dict
def get_items() -> list[dict[str, int]]:
    ...
```

### Use Pyright for Static Type Checking

Prefer **[Pyright](https://github.com/microsoft/pyright)** (or its distribution **basedpyright**) over mypy. Pyright is faster, ships natively in VS Code via the Pylance extension, and has excellent support for modern Python type features. Most editors have a native Pyright integration — use it instead of running a separate type-checker CLI in most workflows.

```bash
# Install standalone (or use your editor's built-in Pylance/Pyright)
uv add --dev pyright
pyright src/
```

`pyproject.toml`:

```toml
[tool.pyright]
include = ["src"]
strict = ["src"]
pythonVersion = "3.11"
```

> **Note:** If you need CI type checking, run `pyright` (or `basedpyright`) in your pipeline. Avoid adding mypy as a second type checker — pick one and be consistent.

### Useful `typing` Constructs

```python
from typing import TypeVar, Protocol, TypeAlias

# TypeAlias for complex types
UserId: TypeAlias = int

# Protocol for structural subtyping (duck typing with type safety)
class Serializable(Protocol):
    def to_dict(self) -> dict[str, object]: ...

# TypeVar for generic functions
T = TypeVar("T")

def first(items: list[T]) -> T | None:
    return items[0] if items else None
```

### `TypedDict` — Typed Dictionary Structures

Use `TypedDict` when you need to type a plain dictionary with a known set of keys, for example when working with JSON payloads, configuration blobs, or legacy APIs that return dicts.

```python
from typing import TypedDict, NotRequired

class Movie(TypedDict):
    title: str
    year: int
    rating: NotRequired[float]  # optional key (Python 3.11+; use total=False in older versions)

# Good — type checker knows the shape
def display(movie: Movie) -> str:
    return f"{movie['title']} ({movie['year']})"

# The checker will flag unknown keys or wrong value types:
bad: Movie = {"title": "Dune", "year": "2021"}  # error: year must be int
```

When to prefer `TypedDict` vs alternatives:

| Use case | Recommended type |
|---|---|
| Typed plain dict (JSON, config) | `TypedDict` |
| Immutable record with logic | `dataclass` / `@dataclass(frozen=True)` |
| External data validation | `pydantic.BaseModel` |
| Simple named tuple | `NamedTuple` |

> **Tip:** `TypedDict` is structural — a plain `dict` with the right keys is assignable. This makes it ideal for typing data at API boundaries without requiring callers to import your class.

### `Literal` — Value-Constrained Types

Use `Literal` to restrict a parameter or return value to a finite set of specific values. This lets the type checker catch invalid arguments and enables exhaustive narrowing.

```python
from typing import Literal

SortOrder = Literal["asc", "desc"]
HttpMethod = Literal["GET", "POST", "PUT", "DELETE", "PATCH"]

# Good — invalid values are caught at type-check time
def sort_users(order: SortOrder) -> list[User]:
    ...

sort_users("asc")    # OK
sort_users("random") # error: Argument of type "random" not assignable to "SortOrder"


# Combine with unions for exhaustive branching
Status = Literal["pending", "active", "archived"]

def handle(status: Status) -> str:
    if status == "pending":
        return "waiting"
    elif status == "active":
        return "running"
    elif status == "archived":
        return "done"
    # Pyright will warn if a branch is missing
```

> **Avoid** using `str` for arguments that only accept a handful of values — `Literal` gives you documentation, IDE completion, and type safety for free.

### `NewType` — Domain-Modelled Primitives

`NewType` creates a distinct named type at type-check time with **zero runtime cost**. Use it to prevent accidentally mixing semantically different values that share the same underlying type.

```python
from typing import NewType

UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)

def get_user(user_id: UserId) -> User: ...
def get_order(order_id: OrderId) -> Order: ...

uid = UserId(42)
oid = OrderId(42)

get_user(uid)   # OK
get_user(oid)   # error: OrderId is not assignable to UserId
get_user(42)    # error: plain int is not assignable to UserId
```

`NewType` is cheaper than a full wrapper class — the constructor is a no-op at runtime (returns the value unchanged), so there is no overhead.

> **When not to use it:** If you need extra methods or validation on the type, use a proper class or `dataclass` instead. `NewType` is purely a type-checker construct.

### `Final` and `ClassVar` — Immutability & Class-Level Annotations

Use `Final` to declare values that must not be reassigned, and `ClassVar` to annotate attributes shared across all instances of a class (as opposed to per-instance attributes).

```python
from typing import Final, ClassVar

# Module-level constant — reassignment is a type error
MAX_RETRIES: Final = 3
MAX_RETRIES = 5  # error: Cannot assign to final variable

# Class-level shared attribute vs instance attribute
class Counter:
    count: ClassVar[int] = 0   # shared across all instances
    name: str                  # per-instance

    def __init__(self, name: str) -> None:
        Counter.count += 1
        self.name = name

# Final instance attribute — set once in __init__, never changed after
class Config:
    def __init__(self, dsn: str) -> None:
        self.dsn: Final = dsn

cfg = Config("postgresql://localhost/db")
cfg.dsn = "other"  # error: Cannot assign to final variable
```

> **Tip:** `Final` on a class attribute without `ClassVar` means each instance gets its own final value (set once in `__init__`). Combine both — `ClassVar[Final[int]]` — for a class-level constant.

### `@overload` — Multiple Call Signatures

Use `@overload` when a function's return type varies depending on the types of its arguments. This gives callers precise type information without losing type safety.

```python
from typing import overload

@overload
def parse(raw: str) -> dict[str, object]: ...
@overload
def parse(raw: bytes) -> dict[str, object]: ...
@overload
def parse(raw: None) -> None: ...

def parse(raw: str | bytes | None) -> dict[str, object] | None:
    """Actual implementation — not type-checked by callers."""
    if raw is None:
        return None
    data = raw if isinstance(raw, str) else raw.decode()
    return json.loads(data)

# The checker now knows the exact return type at each call site:
result: dict[str, object] = parse("{}") # OK
nothing: None = parse(None)             # OK
```

Rules for `@overload`:
- Declare all overloads **before** the implementation.
- The implementation signature must be broad enough to accept all overload cases.
- Only the overload stubs are visible to type checkers — the implementation is hidden.

> **Avoid** using `@overload` just to avoid writing a union type. Reserve it for cases where the *return type* genuinely differs based on *argument types*.

### `TypeGuard` and Type Narrowing

Python's type checker narrows types automatically inside `isinstance` checks. For custom predicates that perform the same narrowing, use `TypeGuard`.

```python
from typing import TypeGuard

# Built-in narrowing — no extra effort needed
def process(value: int | str) -> None:
    if isinstance(value, int):
        print(value + 1)   # value is int here
    else:
        print(value.upper()) # value is str here


# Custom TypeGuard — teach the checker about your own predicate
def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

items: list[object] = ["a", "b", "c"]
if is_string_list(items):
    items[0].upper()  # OK — narrowed to list[str] inside this block
```

Use `assert_never` for exhaustive `match`/`if-elif` chains to ensure every case is handled:

```python
from typing import assert_never, Literal

Status = Literal["pending", "active", "archived"]

def handle(status: Status) -> str:
    if status == "pending":
        return "queued"
    elif status == "active":
        return "running"
    elif status == "archived":
        return "done"
    else:
        assert_never(status)  # type error if a Literal variant is missing
```

### `Self` — Fluent Interfaces and Subclass-Safe Returns

Use `Self` (from `typing`, Python 3.11+; or `typing_extensions` for older versions) when a method returns `self` or `cls`. This ensures subclasses inherit the correct return type without needing a `TypeVar` boilerplate.

```python
from typing import Self

class QueryBuilder:
    def __init__(self) -> None:
        self._filters: list[str] = []

    def where(self, condition: str) -> Self:
        self._filters.append(condition)
        return self

    def limit(self, n: int) -> Self:
        self._limit = n
        return self

class UserQueryBuilder(QueryBuilder):
    def active_only(self) -> Self:
        return self.where("active = true")

# The checker correctly infers UserQueryBuilder, not QueryBuilder:
result: UserQueryBuilder = UserQueryBuilder().active_only().limit(10)
```

Also use `Self` on `@classmethod` factory methods:

```python
class Model:
    @classmethod
    def from_dict(cls, data: dict[str, object]) -> Self:
        obj = cls()
        # populate fields ...
        return obj
```

> **Before `Self`:** The idiomatic workaround was `T = TypeVar("T", bound="QueryBuilder")` on every method — verbose and error-prone in inheritance hierarchies. `Self` eliminates the boilerplate.

### `TYPE_CHECKING` Guard — Avoiding Circular Imports

Circular imports occur when two modules reference each other at runtime. If the import is only needed for type annotations, move it behind a `TYPE_CHECKING` guard — it is `False` at runtime (so the import never executes) but `True` during type-checking.

```python
# models.py
from __future__ import annotations  # makes all annotations lazy strings
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myapp.services import UserService  # only imported when type-checking

class User:
    def process(self, service: UserService) -> None:  # annotation is a string at runtime
        ...
```

```python
# services.py
from myapp.models import User  # no circular import — models.py never imports services at runtime

class UserService:
    def activate(self, user: User) -> None:
        ...
```

Key points:
- Always pair `TYPE_CHECKING` imports with `from __future__ import annotations` (or quote the annotation manually: `"UserService"`).
- Only use this for type-annotation-only imports — if you call methods on the imported object at runtime, you need a real import.
- Prefer restructuring the module graph if circular imports become pervasive; `TYPE_CHECKING` is a workaround, not a design pattern.

### `Annotated` — Metadata on Types

`Annotated[T, metadata]` attaches arbitrary metadata to a type without changing its static type. Frameworks like Pydantic v2 and FastAPI use this to attach validation rules, documentation, and dependency injection hints directly to type annotations.

```python
from typing import Annotated
from pydantic import BaseModel, Field, Gt, Lt

# Pydantic v2: validation rules embedded in the type
class Order(BaseModel):
    quantity: Annotated[int, Gt(0)]             # must be > 0
    discount: Annotated[float, Ge(0), Le(1)]    # 0.0 – 1.0
    description: Annotated[str, Field(max_length=200)]


# FastAPI: dependency injection via Annotated
from fastapi import Depends, FastAPI

app = FastAPI()

def get_db() -> Database: ...

DatabaseDep = Annotated[Database, Depends(get_db)]  # reusable alias

@app.get("/users")
def list_users(db: DatabaseDep) -> list[User]:
    return db.query(User).all()
```

You can also define your own metadata for documentation or custom tooling:

```python
from dataclasses import dataclass
from typing import Annotated

@dataclass
class Positive:
    """Marker: value must be > 0."""

Quantity = Annotated[int, Positive()]  # reusable semantic alias

def set_stock(amount: Quantity) -> None: ...
```

> **Tip:** `Annotated` metadata is ignored by the default type checker — it only matters to frameworks and tools that explicitly read `__metadata__`. The static type is always the first argument (`T`).

---

## 3. Project Structure & Packaging

### Prefer the `src/` Layout

Using a `src/` layout prevents accidental imports of the uninstalled package during development.

```
my_project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   └── test_core.py
├── pyproject.toml
└── README.md
```

### Use `pyproject.toml` as the Single Source of Truth

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-package"
version = "0.1.0"
description = "Short description"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

### Keep `__init__.py` Minimal
Expose only the public API; avoid heavy logic or side effects in `__init__.py`.

---

## 4. Dependency Management

### Use `uv` — The Recommended Dependency Manager

**[uv](https://github.com/astral-sh/uv)** is the strongly recommended tool for all Python dependency and environment management. It is a single, extremely fast binary (written in Rust) that replaces `pip`, `pip-tools`, `venv`, `virtualenv`, and more. Unless you have a strong reason to use something else, use `uv` for every project.

```bash
# Install uv (one-time, system-wide)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a new project with uv
uv init my-project
cd my-project

# Create and manage virtual environments
uv venv                        # create .venv
source .venv/bin/activate      # activate (Linux/macOS)

# Add/remove dependencies (updates pyproject.toml automatically)
uv add httpx pydantic
uv add --dev pytest ruff pyright
uv remove httpx

# Install all dependencies from pyproject.toml
uv sync                        # install exact locked versions
uv sync --all-extras           # include optional dependency groups

# Run tools without activating the venv
uv run pytest tests/
uv run ruff check .
```

`pyproject.toml` with uv:

```toml
[project]
name = "my-package"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[dependency-groups]
dev = ["pytest>=8", "ruff>=0.4", "pyright>=1.1"]
```

`uv` also generates a `uv.lock` file — commit this to version control for reproducible installs across environments.

> **Avoid:** For new projects, avoid using bare `pip` + `requirements.txt` workflows. They lack dependency locking and environment management that `uv` provides automatically.

### Pin Dependencies in Applications, Range-Constrain in Libraries

```toml
# Application: pin to exact versions for reproducibility (uv.lock handles this)
dependencies = ["django==5.0.4", "psycopg[binary]==3.1.19"]

# Library: use ranges to avoid conflicts with dependents
dependencies = ["httpx>=0.25,<1.0"]
```

### Use a `.gitignore` Appropriate for Python

A good `.gitignore` prevents committing virtual environments, build artefacts, cache files, and editor metadata. **Always start from [GitHub's official Python `.gitignore` template](https://github.com/github/gitignore/blob/main/Python.gitignore)** — it covers the full spectrum of Python tooling noise.

Key entries you must have:

```gitignore
# Virtual environments
.venv/
venv/
env/
ENV/

# Byte-compiled / optimisation cache
__pycache__/
*.py[cod]
*$py.class

# Distribution / packaging
dist/
build/
*.egg-info/
*.egg
MANIFEST

# uv
uv.lock is committed to VCS; the following are NOT:
.python-version      # if managed per-project

# Type checker caches
.mypy_cache/
.pyright/

# Ruff / linter caches
.ruff_cache/

# Test & coverage artefacts
.pytest_cache/
.coverage
htmlcov/
coverage.xml

# Environment variable files — NEVER commit secrets
.env
.env.*
!.env.example      # it's fine to commit a documented example file

# Editor / IDE files
.idea/
.vscode/
*.swp
```

> **Tip:** When starting a new repo on GitHub, select the **Python** template from the `.gitignore` dropdown — this gives you the community-maintained list maintained by GitHub. Revisit and prune it as your toolchain evolves.

---

## 5. Testing

### Use pytest

```bash
uv add --dev pytest pytest-cov
uv run pytest tests/ -v --cov=src/my_package --cov-report=term-missing
```

### Structure Tests Clearly

```python
# tests/test_core.py
import pytest
from my_package.core import divide

class TestDivide:
    def test_positive_numbers(self):
        assert divide(10, 2) == 5.0

    def test_negative_divisor(self):
        assert divide(-6, 3) == -2.0

    def test_division_by_zero_raises(self):
        with pytest.raises(ZeroDivisionError):
            divide(1, 0)
```

### Use Fixtures for Shared Setup

```python
import pytest
from my_package.db import Database

@pytest.fixture
def db():
    database = Database(":memory:")
    database.migrate()
    yield database
    database.close()

def test_insert_user(db):
    db.insert_user(name="Alice")
    assert db.count_users() == 1
```

### Aim for High Coverage, Not 100%

- Target **80–90%** code coverage as a practical goal
- Prioritise testing business logic and edge cases over trivial getters/setters
- Use `# pragma: no cover` sparingly for unreachable or platform-specific branches

### Parametrize to Reduce Duplication

```python
@pytest.mark.parametrize("value,expected", [
    (0, "zero"),
    (1, "positive"),
    (-1, "negative"),
])
def test_classify(value, expected):
    assert classify(value) == expected
```

---

## 6. Documentation

### Write Docstrings for All Public Symbols

Use **Google-style** docstrings (readable, well-supported by tooling):

```python
def retry(func, *, max_attempts: int = 3, delay: float = 1.0):
    """Retry a callable on failure with exponential back-off.

    Args:
        func: The callable to retry.
        max_attempts: Maximum number of attempts before raising.
        delay: Initial delay in seconds between retries.

    Returns:
        The return value of ``func`` on success.

    Raises:
        Exception: Re-raises the last exception after exhausting retries.

    Example:
        >>> retry(lambda: requests.get("https://example.com"), max_attempts=5)
    """
```

### Keep Docstrings and Code in Sync
Outdated documentation is worse than no documentation. Update docstrings when you change behaviour.

### Use `mkdocs` for Project Docs

**[MkDocs](https://www.mkdocs.org/)** with the [Material theme](https://squidfunk.github.io/mkdocs-material/) is the recommended way to turn Markdown files into a polished documentation site. It's simple to configure, renders beautifully, and integrates with GitHub Pages for free hosting.

```bash
uv add --dev mkdocs mkdocs-material mkdocstrings[python]
mkdocs new .       # scaffold mkdocs.yml and docs/index.md
mkdocs serve       # live-reload dev server at http://127.0.0.1:8000
mkdocs build       # build static site into site/
mkdocs gh-deploy   # publish to GitHub Pages
```

Minimal `mkdocs.yml`:

```yaml
site_name: My Project
theme:
  name: material
plugins:
  - mkdocstrings:
      handlers:
        python:
          paths: [src]
nav:
  - Home: index.md
  - API Reference: api.md
```

---

## 7. Error Handling & Logging

### Be Specific with Exceptions

```python
# Good — specific, recoverable
try:
    result = json.loads(raw)
except json.JSONDecodeError as exc:
    logger.warning("Invalid JSON payload: %s", exc)
    return None

# Avoid — swallows everything including KeyboardInterrupt
try:
    result = json.loads(raw)
except:
    return None
```

### Define Custom Exception Hierarchies

```python
class AppError(Exception):
    """Base class for all application errors."""

class ConfigError(AppError):
    """Raised when configuration is missing or invalid."""

class NetworkError(AppError):
    """Raised on network-related failures."""
```

### Use `logging`, Not `print`

Replace `print()` calls with the `logging` module for production code — it adds severity levels, filtering, and structured output. See the dedicated **[Logging Guide](../docs/logging.md)** for logger naming, configuration patterns, structured logging with `structlog` / `python-json-logger`, and a full worked example.

---

## 8. Performance

### Profile Before Optimising

```python
import cProfile
import pstats

with cProfile.Profile() as pr:
    my_function()

stats = pstats.Stats(pr)
stats.sort_stats("cumulative").print_stats(20)
```

Or use `py-spy` for low-overhead sampling on live processes:

```bash
uv tool install py-spy
py-spy top --pid <PID>
```

### Prefer Comprehensions Over Loops for Simple Transforms

```python
# Good
squares = [x * x for x in range(100)]
even_map = {k: v for k, v in data.items() if v % 2 == 0}

# Avoid for simple cases
squares = []
for x in range(100):
    squares.append(x * x)
```

### Use Generators for Large Sequences

```python
# Memory-efficient: processes one item at a time
def read_large_file(path: str):
    with open(path) as f:
        for line in f:
            yield line.strip()

# Avoid loading everything into memory
lines = list(open("huge.log").readlines())
```

### Cache Expensive, Pure Computations

```python
from functools import lru_cache, cache

@cache  # unbounded (Python 3.9+)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

@lru_cache(maxsize=128)  # bounded cache
def fetch_config(env: str) -> dict:
    ...
```

### Use `__slots__` for Data-Heavy Classes

By default, every Python instance stores its attributes in a `__dict__` — a regular dictionary that's flexible but uses significant memory. **`__slots__`** replaces this per-instance dictionary with a fixed set of named slots, which are stored more compactly. For classes that are instantiated in large numbers (e.g. data records, nodes in a graph, geometric primitives), this can cut memory usage by 30–50 % and slightly improve attribute-access speed.

```python
# Without __slots__: each instance carries a full dict (~200 bytes overhead)
class PointDict:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

# With __slots__: only the two declared slots are allocated (~56 bytes overhead)
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

Trade-offs to be aware of:
- You **cannot add arbitrary attributes** to an instance at runtime (no dynamic `__dict__`).
- **Multiple inheritance** with `__slots__` requires care — all base classes must also declare `__slots__` (or use `__dict__`) or the benefit is lost.
- For most classes `__slots__` is unnecessary; use it only when profiling shows memory is a bottleneck.

> **Modern alternative:** [`dataclasses`](https://docs.python.org/3/library/dataclasses.html) support `__slots__` via `@dataclass(slots=True)` (Python 3.10+), which is cleaner than declaring them manually.

---

## 9. Security

### Never Hardcode Secrets

```python
# Bad
API_KEY = "sk-live-abc123"

# Good — read from environment
import os
API_KEY = os.environ["API_KEY"]

# Better — use python-dotenv in development
from dotenv import load_dotenv
load_dotenv()
API_KEY = os.environ["API_KEY"]
```

### Avoid `eval()` and `exec()` on Untrusted Input

```python
# Dangerous
user_input = "__import__('os').system('rm -rf /')"
eval(user_input)  # Never do this

# Use ast.literal_eval for safe evaluation of literals
import ast
value = ast.literal_eval("[1, 2, 3]")
```

### Use Parameterised Queries for Databases

```python
# Vulnerable to SQL injection
query = f"SELECT * FROM users WHERE name = '{name}'"

# Safe — use parameterised queries
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))
```

### Validate and Sanitise Input with Pydantic

```python
from pydantic import BaseModel, EmailStr, field_validator

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr
    age: int

    @field_validator("age")
    @classmethod
    def age_must_be_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("Age must be positive")
        return v
```

### Scan Dependencies for Vulnerabilities

```bash
uv tool install pip-audit
pip-audit
```

---

## 10. Common Anti-Patterns to Avoid

### Mutable Default Arguments

```python
# Bug: all callers share the same list object
def append_item(item, lst=[]):
    lst.append(item)
    return lst

# Fix: use None and create a new list inside
def append_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

### Bare `except` Clauses

```python
# Swallows KeyboardInterrupt, SystemExit, MemoryError, etc.
try:
    risky()
except:
    pass

# Be explicit
try:
    risky()
except ValueError as exc:
    logger.warning("Expected error: %s", exc)
```

### Comparing to `None` / `True` / `False` with `==`

```python
# Bad
if result == None:
    ...
if flag == True:
    ...

# Good
if result is None:
    ...
if flag:
    ...
```

### Shadowing Built-ins

```python
# Avoid naming variables after built-ins
list = [1, 2, 3]    # shadows built-in list()
input = "hello"     # shadows built-in input()
id = 42             # shadows built-in id()
```

### Overusing `*` Imports

```python
# Pollutes namespace, breaks static analysis
from os.path import *

# Be explicit
from os.path import join, exists, dirname
```

### Not Using Context Managers for Resources

```python
# Risky: file not closed if exception occurs
f = open("data.txt")
data = f.read()
f.close()

# Correct: always use context managers
with open("data.txt") as f:
    data = f.read()
```

### String Concatenation in Loops

```python
# O(n²) due to string immutability — slow for large n
result = ""
for word in words:
    result += word + " "

# Correct: collect and join once
result = " ".join(words)
```

---

## Further Reading

- [PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [PEP 484 – Type Hints](https://peps.python.org/pep-0484/)
- [PEP 544 – Protocols: Structural Subtyping](https://peps.python.org/pep-0544/)
- [PEP 586 – Literal Types](https://peps.python.org/pep-0586/)
- [PEP 589 – TypedDict](https://peps.python.org/pep-0589/)
- [PEP 591 – Final qualifier](https://peps.python.org/pep-0591/)
- [PEP 647 – User-Defined Type Guards](https://peps.python.org/pep-0647/)
- [PEP 673 – Self Type](https://peps.python.org/pep-0673/)
- [PEP 593 – Annotated Types](https://peps.python.org/pep-0593/)
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Effective Python (3rd Edition) — Brett Slatkin](https://effectivepython.com/)
- [Python Docs — Logging HOWTO](https://docs.python.org/3/howto/logging.html)
- [Real Python — Python Best Practices](https://realpython.com/tutorials/best-practices/)
- [mypy docs — Type narrowing](https://mypy.readthedocs.io/en/stable/type_narrowing.html)
