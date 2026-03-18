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
pip install ruff
ruff format .      # format files
ruff check .       # lint

# Alternative: Black + isort
pip install black isort
black .
isort .
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

### Run mypy in Strict Mode

```bash
pip install mypy
mypy --strict src/
```

`pyproject.toml`:

```toml
[tool.mypy]
strict = true
ignore_missing_imports = true
```

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

### Use a Modern Dependency Manager

| Tool | Best For |
|------|---------|
| [uv](https://github.com/astral-sh/uv) | Speed-focused, drop-in pip/venv replacement |
| [Poetry](https://python-poetry.org/) | Full dependency resolution + packaging |
| [pip + venv](https://docs.python.org/3/library/venv.html) | Minimal, built-in, always available |

### Pin Dependencies in Applications, Range-Constrain in Libraries

```toml
# Application: pin to exact versions for reproducibility
dependencies = ["django==5.0.4", "psycopg[binary]==3.1.19"]

# Library: use ranges to avoid conflicts with dependents
dependencies = ["httpx>=0.25,<1.0"]
```

### Never Commit Virtual Environments
Add to `.gitignore`:
```
.venv/
venv/
__pycache__/
*.pyc
.mypy_cache/
.ruff_cache/
```

### Use `uv` for Fast Workflows

```bash
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"
uv pip sync requirements.txt   # fast reproducible installs
```

---

## 5. Testing

### Use pytest

```bash
pip install pytest pytest-cov
pytest tests/ -v --cov=src/my_package --cov-report=term-missing
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

### Use `mkdocs` or `sphinx` for Project Docs

```bash
# MkDocs + Material theme (recommended for simplicity)
pip install mkdocs mkdocs-material mkdocstrings[python]
mkdocs new .
mkdocs serve
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

```python
import logging

logger = logging.getLogger(__name__)   # per-module logger

# In your app entry point, configure once:
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)

# Usage
logger.debug("Processing item %d", item_id)
logger.info("Server started on port %d", port)
logger.error("Failed to connect to DB: %s", exc, exc_info=True)
```

### Prefer Structured Logging in Production

Use [`structlog`](https://www.structlog.org/) or [`python-json-logger`](https://github.com/madzak/python-json-logger) for machine-parseable logs:

```python
import structlog

log = structlog.get_logger()
log.info("user_created", user_id=42, email="alice@example.com")
# {"event": "user_created", "user_id": 42, "email": "alice@example.com", ...}
```

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
pip install py-spy
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

### Use `slots` for Data-Heavy Classes

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

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
pip install pip-audit
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
- [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)
- [Effective Python (3rd Edition) — Brett Slatkin](https://effectivepython.com/)
- [Python Docs — Logging HOWTO](https://docs.python.org/3/howto/logging.html)
- [Real Python — Python Best Practices](https://realpython.com/tutorials/best-practices/)
