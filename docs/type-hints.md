# Python Type Hints Best Practices

A comprehensive guide to writing expressive, correct, and maintainable type annotations in Python.

---

## Table of Contents

1. [Why Type Hints Matter](#1-why-type-hints-matter)
2. [Basic Annotation Syntax](#2-basic-annotation-syntax)
3. [Modern Type Syntax (Python 3.9 / 3.10+)](#3-modern-type-syntax-python-39--310)
4. [`typing` Module Essentials](#4-typing-module-essentials)
5. [Protocols & Structural Subtyping](#5-protocols--structural-subtyping)
6. [Generics and TypeVar](#6-generics-and-typevar)
7. [Runtime vs. Static Typing](#7-runtime-vs-static-typing)
8. [Static Analysis with Pyright](#8-static-analysis-with-pyright)
9. [Annotating Special Cases](#9-annotating-special-cases)
10. [Common Anti-Patterns](#10-common-anti-patterns)

---

## 1. Why Type Hints Matter

Python is dynamically typed — you never *have* to annotate a variable or function. So why bother?

**Better tooling.** IDEs (VS Code, PyCharm) and language servers use type information to provide accurate autocompletion, inline documentation, and real-time error highlighting. Without annotations, tools fall back to guessing.

**Earlier bug detection.** A static type checker running in CI catches entire classes of bugs — wrong argument order, missing return values, `None`-dereference errors — before they ever reach a test environment.

**Self-documenting code.** A function signature like `def send(recipient: str, message: str, *, retries: int = 3) -> bool` tells you everything you need to call it correctly. No docstring archaeology required.

**Safer refactoring.** When you rename a field or change a return type, a type checker immediately surfaces every call site that needs updating.

Type hints are *gradually typed*: you can annotate as little or as much as you like, and unannotated code remains valid Python. The practical approach is to annotate all public APIs first, then incrementally add coverage to internals.

---

## 2. Basic Annotation Syntax

### Variable annotations

```python
# Simple variable annotations
name: str = "Alice"
count: int = 0
ratio: float = 0.75
active: bool = True

# Annotate without assignment (declares the type, no value yet)
user_id: int
```

### Function annotations

Annotate every parameter and the return type. Use `-> None` explicitly when a function returns nothing — it signals intent and prevents accidental `None` returns from going unnoticed.

```python
def greet(name: str) -> str:
    return f"Hello, {name}"

def save_record(record: dict[str, object]) -> None:
    ...

# Keyword-only arguments work the same way
def connect(host: str, port: int, *, timeout: float = 30.0) -> bool:
    ...
```

### Annotate all public APIs

Type-check everything that crosses a module boundary. Internal helpers can be annotated opportunistically, but public functions, methods, and class attributes should always be fully annotated.

```python
# Good — callers immediately know the contract
def fetch_user(user_id: int, *, active_only: bool = True) -> "User | None":
    ...

# Avoid — forces callers to read the implementation
def fetch_user(user_id, active_only=True):
    ...
```

---

## 3. Modern Type Syntax (Python 3.9 / 3.10+)

Python 3.9 and 3.10 introduced cleaner built-in syntax for common type patterns. Prefer these over the legacy `typing` equivalents in all new code.

### Use built-in generic collections (3.9+)

```python
# Prefer (3.9+)
def get_scores() -> list[int]: ...
def get_config() -> dict[str, str]: ...
def get_pair() -> tuple[str, int]: ...
def get_tags() -> set[str]: ...

# Avoid (legacy — only needed for Python < 3.9)
from typing import List, Dict, Tuple, Set
def get_scores() -> List[int]: ...
```

### Use `X | Y` union syntax (3.10+)

```python
# Prefer (3.10+)
def process(value: int | str | None) -> list[str]: ...

# Avoid (legacy)
from typing import List, Optional, Union
def process(value: Union[int, str, None]) -> List[str]: ...

# Optional[X] is just shorthand for X | None — avoid it in new code
def find(name: str) -> str | None: ...   # clear and explicit
```

### `from __future__ import annotations`

Adding `from __future__ import annotations` at the top of a module **postpones the evaluation** of all annotations, turning them into strings at runtime rather than evaluating them eagerly. This has two practical uses:

- **Forward references without quotes** — you can refer to a class before it is defined without wrapping the name in a string.
- **Avoiding import-time costs** — annotations are only resolved when explicitly requested (e.g., via `typing.get_type_hints()`).

> **Important:** This import does *not* unlock the `X | Y` union syntax on Python 3.7–3.9. The `|` operator for types requires the Python 3.10+ parser. On 3.7–3.9, use `typing.Union[X, Y]` and `typing.Optional[X]` for unions.

```python
from __future__ import annotations


# Forward reference: Node refers to itself before the class is fully defined
class Node:
    def __init__(self, next: Node | None = None) -> None:  # no quotes needed
        self.next = next


# On Python 3.7–3.9, union types still require typing.Union / typing.Optional:
from typing import Optional, Union

def find(name: str) -> Optional[str]: ...          # 3.7–3.9 safe
def process(value: Union[int, str]) -> str: ...    # 3.7–3.9 safe
```

> **Caveat:** Deferred evaluation breaks code that inspects `__annotations__` at runtime (e.g., Pydantic v1, some dataclass utilities). On Python 3.10+ you rarely need this import — the native syntax already handles forward references via lazy evaluation.

---

## 4. `typing` Module Essentials

While modern syntax has replaced many `typing` exports, several remain essential.

### `TypeAlias` — name complex types

```python
from typing import Callable, TypeAlias

# Give a meaningful name to a complex type
Headers: TypeAlias = dict[str, str]
Callback: TypeAlias = Callable[[int, str], bool]
Matrix: TypeAlias = list[list[float]]

def send_request(url: str, headers: Headers) -> None: ...
```

### `Callable` — annotate functions as values

```python
from typing import Callable, TypeAlias

# Callable[[arg_types...], return_type]
Handler: TypeAlias = Callable[[str, int], bool]

def register(event: str, handler: Callable[[str], None]) -> None: ...

# No-argument callable returning nothing
Thunk: TypeAlias = Callable[[], None]
```

### `ClassVar` — class-level attributes

```python
from typing import ClassVar

class Config:
    MAX_RETRIES: ClassVar[int] = 3   # belongs to the class, not instances
    _instances: ClassVar[list["Config"]] = []

    timeout: float   # instance attribute
```

### `Final` — constants and sealed values

```python
from typing import Final

MAX_SIZE: Final = 1024
BASE_URL: Final[str] = "https://api.example.com"

# Final on a class prevents subclassing
from typing import final

@final
class Singleton:
    ...
```

### `Literal` — restrict to specific values

```python
from typing import Literal

Direction: TypeAlias = Literal["north", "south", "east", "west"]
LogLevel: TypeAlias = Literal["DEBUG", "INFO", "WARNING", "ERROR"]

def move(direction: Direction, steps: int = 1) -> None: ...

def set_level(level: LogLevel) -> None: ...
```

### `TypedDict` — typed dictionaries

Use `TypedDict` when you must work with dictionaries that have a fixed, known shape (e.g., JSON payloads, config blobs). Prefer dataclasses or Pydantic models for new code you control.

```python
from typing import TypedDict, Required, NotRequired

class UserRecord(TypedDict):
    id: int
    name: str
    email: str
    role: NotRequired[str]   # optional key

def create_user(record: UserRecord) -> None: ...
```

### `overload` — multiple call signatures

```python
from typing import overload

@overload
def parse(value: str) -> int: ...
@overload
def parse(value: bytes) -> str: ...

def parse(value: str | bytes) -> int | str:
    if isinstance(value, str):
        return int(value)
    return value.decode()
```

---

## 5. Protocols & Structural Subtyping

`Protocol` enables **duck typing with type safety**: define the interface you need, and any class that implements those methods satisfies the protocol — no inheritance required. This is Python's approach to Go-style interfaces.

```python
from typing import Protocol, runtime_checkable

class Serializable(Protocol):
    def to_dict(self) -> dict[str, object]: ...

class Closeable(Protocol):
    def close(self) -> None: ...

# Any object with a .to_dict() method satisfies Serializable
class User:
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age

    def to_dict(self) -> dict[str, object]:
        return {"name": self.name, "age": self.age}

def persist(obj: Serializable) -> None:
    data = obj.to_dict()
    ...

persist(User("Alice", 30))   # type-checks correctly, no ABC needed
```

### `runtime_checkable` protocols

Add `@runtime_checkable` to allow `isinstance()` checks at runtime — useful for validation code.

```python
@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None:
        print("○")

assert isinstance(Circle(), Drawable)   # True at runtime
```

> **Caveat:** `runtime_checkable` only checks for method *existence*, not signature. Use it for defensive checks, not as a substitute for proper tests.

### Prefer Protocols over ABCs for external types

Use `Protocol` when you want to accept any object that *happens* to have the right methods — especially for third-party types you can't modify. Use ABCs when you own the class hierarchy and want to enforce implementation.

```python
# Good — accepts any file-like object without requiring inheritance
class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

def process_stream(stream: Readable) -> bytes:
    return stream.read()

# Works with open(), io.BytesIO(), socket.makefile(), etc.
```

---

## 6. Generics and TypeVar

Generics let you write functions and classes that are type-safe across different types without sacrificing flexibility.

### `TypeVar` — generic functions

```python
from typing import TypeVar

T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

def first(items: list[T]) -> T | None:
    return items[0] if items else None

# The return type is inferred from the argument type
result: int | None = first([1, 2, 3])   # T = int
name: str | None = first(["a", "b"])    # T = str
```

### Bounded TypeVar — constrain the type

```python
from typing import Any, Protocol, TypeVar


# Define the structural bound as a Protocol
class SupportsLessThan(Protocol):
    def __lt__(self, other: Any) -> bool: ...


# Union constraint: T must be exactly int or float (not a subtype of either)
Numeric = TypeVar("Numeric", int, float)

# Upper bound: T must be a subtype of SupportsLessThan (i.e. support <)
Comparable = TypeVar("Comparable", bound=SupportsLessThan)


def maximum(a: Comparable, b: Comparable) -> Comparable:
    return b if a < b else a
```

### Generic classes (Python 3.12+ `type` syntax)

```python
# Python 3.12+ — clean built-in generic syntax
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

    def peek(self) -> T | None:
        return self._items[-1] if self._items else None

# Python 3.9–3.11 — use Generic base class
from typing import Generic

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()
```

### `ParamSpec` and `Concatenate` — decorators that preserve signatures

```python
from typing import ParamSpec, Callable, TypeVar
import functools

P = ParamSpec("P")
R = TypeVar("R")

def logged(func: Callable[P, R]) -> Callable[P, R]:
    """Decorator that logs calls — preserves the wrapped function's signature."""
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logged
def add(x: int, y: int) -> int:   # signature is preserved for type checkers
    return x + y
```

---

## 7. Runtime vs. Static Typing

### Annotations are not enforced at runtime

Python does not validate type annotations at runtime by default. The following code runs without error:

```python
def double(n: int) -> int:
    return n * 2

double("oops")   # runs fine — returns "oopsoops"
```

Use a **static type checker** (Pyright, mypy) in CI to catch these errors before they reach production. If you need *runtime* validation, use Pydantic or `beartype`.

### `get_type_hints()` — access annotations at runtime

```python
import typing

def greet(name: str) -> str:
    return f"Hello, {name}"

hints = typing.get_type_hints(greet)
# {'name': <class 'str'>, 'return': <class 'str'>}
```

Use `get_type_hints()` rather than accessing `__annotations__` directly — it resolves forward references (string annotations) correctly.

### `TYPE_CHECKING` — avoid circular imports

The `TYPE_CHECKING` constant is `False` at runtime and `True` only when a type checker analyses the code. Use it to import types used only in annotations, preventing circular import errors.

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myapp.models import User   # only imported during type checking

def process(user: User) -> None:   # safe: annotation is a string at runtime
    ...
```

### Runtime validation with Pydantic

For data that crosses trust boundaries (API requests, config files, CLI arguments), use Pydantic to validate at runtime with the same annotations you write for type checkers.

```python
from pydantic import BaseModel, EmailStr, PositiveInt

class CreateUserRequest(BaseModel):
    name: str
    email: EmailStr
    age: PositiveInt

# Pydantic validates types and constraints at instantiation time
request = CreateUserRequest(name="Alice", email="alice@example.com", age=30)
```

---

## 8. Static Analysis with Pyright

Prefer **[Pyright](https://github.com/microsoft/pyright)** (or its distribution **basedpyright**) over mypy for new projects. Pyright is faster, has excellent support for modern Python type features, and ships natively in VS Code via the Pylance extension.

### Install and run

```bash
uv add --dev pyright
uv run pyright src/
```

### Configure `pyproject.toml`

```toml
[tool.pyright]
include = ["src"]
exclude = ["**/__pycache__"]
pythonVersion = "3.11"
typeCheckingMode = "standard"   # "off" | "basic" | "standard" | "strict"

# Opt specific files into stricter checking
strict = ["src/myapp/core.py", "src/myapp/models.py"]
```

### Strictness levels

| Mode | What it checks |
|------|---------------|
| `off` | No type checking — annotations are still parsed |
| `basic` | Most common errors; allows unannotated functions |
| `standard` | Recommended default; catches most real bugs |
| `strict` | Requires full annotation coverage; zero tolerance for `Unknown` types |

Start at `standard` for a new project. Add `strict` incrementally to the most critical modules.

### Integrate into CI

```yaml
# .github/workflows/ci.yml (excerpt)
- name: Type check
  run: uv run pyright
```

### Integrate with pre-commit

```yaml
# .pre-commit-config.yaml (excerpt)
- repo: local
  hooks:
    - id: pyright
      name: pyright
      entry: uv run pyright
      language: system
      types: [python]
      pass_filenames: false
```

### Inline suppression

Suppress a specific error on a single line with `# type: ignore[error-code]`. Always include the error code — bare `# type: ignore` is too broad and hides future errors.

```python
# Narrow suppression — preferred
result = legacy_api()  # type: ignore[no-untyped-call]

# Avoid — suppresses everything on this line
result = legacy_api()  # type: ignore
```

---

## 9. Annotating Special Cases

### Dataclasses

`@dataclass` integrates seamlessly with type hints — every field annotation becomes an `__init__` parameter.

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Point:
    x: float
    y: float
    label: str = ""
    tags: list[str] = field(default_factory=list)
    _count: ClassVar[int] = 0   # class variable, not an __init__ param

@dataclass(frozen=True)   # immutable
class Config:
    host: str
    port: int = 8080

@dataclass(slots=True)   # __slots__ automatically (Python 3.10+)
class Pixel:
    r: int
    g: int
    b: int
```

### `NamedTuple` — typed tuples

```python
from typing import NamedTuple

class Coordinate(NamedTuple):
    latitude: float
    longitude: float
    altitude: float = 0.0

coord = Coordinate(51.5, -0.1)
lat: float = coord.latitude   # attribute access, fully typed
```

### `Self` — fluent interfaces and class methods

```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:
        self._name = name
        return self

    def set_value(self, value: int) -> Self:
        self._value = value
        return self

    @classmethod
    def default(cls) -> Self:
        return cls()
```

### `Unpack` and `TypeVarTuple` — variadic generics (3.11+)

```python
from typing import Callable, TypeVarTuple, Unpack

Ts = TypeVarTuple("Ts")

def broadcast(func: Callable[[Unpack[Ts]], None], *args: Unpack[Ts]) -> None:
    func(*args)
```

### Annotating `__init__` and `__new__`

`__init__` always returns `None`. `__new__` returns `Self`. Both are implicit — you rarely need to annotate them explicitly, but Pyright will flag incorrect return annotations.

```python
class MyClass:
    def __init__(self, value: int) -> None:   # always -> None
        self.value = value
```

### Annotating generators and async code

```python
from typing import Generator, AsyncGenerator, Iterator, AsyncIterator

# Synchronous generator: Generator[YieldType, SendType, ReturnType]
def counter(start: int, stop: int) -> Generator[int, None, None]:
    for i in range(start, stop):
        yield i

# Simpler alias when send/return are None
def lines(path: str) -> Iterator[str]:
    with open(path) as f:
        yield from f

# Async generator
async def arange(n: int) -> AsyncGenerator[int, None]:
    for i in range(n):
        yield i

# Async iterator (for __aiter__ / __anext__)
async def fetch_pages(url: str) -> AsyncIterator[bytes]:
    ...
```

---

## 10. Common Anti-Patterns

### Using `Any` as an escape hatch

`Any` disables type checking for a value — it is compatible with every type in both directions. This is sometimes unavoidable (e.g., interfacing with truly dynamic code), but overusing it defeats the purpose of type hints.

```python
import json
from typing import Any

# Avoid — loses all type safety
def process(data: Any) -> Any:
    return data["value"]

# Prefer — even a broad type is better than Any
def process(data: dict[str, object]) -> object:
    return data["value"]

# Use Any sparingly, document why
def parse_legacy_response(raw: bytes) -> Any:
    # Legacy API returns inconsistent shapes; typed in follow-up issue #42
    return json.loads(raw)
```

### Bare `object` vs. `Any`

`object` is the base of all Python classes — every type is a subtype of `object`. Unlike `Any`, it is *not* compatible upward: you cannot call arbitrary methods on an `object`. Prefer `object` over `Any` when you mean "I don't care what type this is, but I won't use it".

```python
def log_value(value: object) -> None:
    print(repr(value))   # repr() works on all objects — fine
    value.strip()        # Error: object has no attribute strip — caught!
```

### Annotating with string literals unnecessarily

Forward reference strings (`"ClassName"`) were necessary in older Python to break circular references. With `from __future__ import annotations` or Python 3.10+, use bare names instead.

```python
from __future__ import annotations

# Good (with the import above)
class Node:
    def __init__(self, next: Node | None = None) -> None:
        self.next = next

# Avoid — redundant quotes
class Node:
    def __init__(self, next: "Node | None" = None) -> None:
        self.next = next
```

### Ignoring `None` in return types

A function that can return `None` must declare it. Omitting `None` from the return type causes subtle bugs and misleading IDE completions.

```python
# Bug: callers think this always returns a User
def find_user(user_id: int) -> User:
    if user_id in self._cache:
        return self._cache[user_id]
    # implicitly returns None — Pyright will flag this

# Correct
def find_user(user_id: int) -> User | None:
    ...
```

### Using mutable defaults in dataclass fields

```python
# Bug: all instances share the same list
@dataclass
class Bag:
    items: list[str] = []   # Pyright/mypy will flag this

# Correct
@dataclass
class Bag:
    items: list[str] = field(default_factory=list)
```

### Narrowing with `assert` instead of `isinstance`

`assert` is stripped in optimised mode (`python -O`). Use `isinstance` guards for narrowing, not assertions.

```python
def process(value: int | str) -> str:
    # Bad: assert stripped in production
    assert isinstance(value, str)
    return value.upper()

    # Good: isinstance guard — narrows the type and is always enforced
    if isinstance(value, str):
        return value.upper()
    return str(value)
```

### Forgetting `-> None` on side-effect functions

A missing return annotation is treated as `-> Unknown` in strict mode. Always annotate functions that return nothing.

```python
# Ambiguous — is None intentional or forgotten?
def cleanup(tmp_dir: str):
    shutil.rmtree(tmp_dir)

# Explicit
def cleanup(tmp_dir: str) -> None:
    shutil.rmtree(tmp_dir)
```

---

## Further Reading

- [PEP 484 – Type Hints](https://peps.python.org/pep-0484/)
- [PEP 526 – Variable Annotations](https://peps.python.org/pep-0526/)
- [PEP 544 – Protocols](https://peps.python.org/pep-0544/)
- [PEP 585 – Built-in Generic Types](https://peps.python.org/pep-0585/)
- [PEP 604 – Union Syntax `X | Y`](https://peps.python.org/pep-0604/)
- [Pyright documentation](https://microsoft.github.io/pyright/)
- [mypy documentation](https://mypy.readthedocs.io/)
- [Python Docs — `typing` module](https://docs.python.org/3/library/typing.html)
- [Python Docs — Generics](https://docs.python.org/3/library/typing.html#generics)
