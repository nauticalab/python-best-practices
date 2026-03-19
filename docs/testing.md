# Python Testing Best Practices

A practical guide to writing reliable, maintainable tests in Python using pytest and the broader testing ecosystem.

---

## Table of Contents

1. [Where to Start](#1-where-to-start)
2. [Unit Tests](#2-unit-tests)
3. [Integration Tests](#3-integration-tests)
4. [End-to-End Tests](#4-end-to-end-tests)
5. [Property-Based Testing](#5-property-based-testing)
6. [Mocking & Patching](#6-mocking--patching)
7. [Test Organisation](#7-test-organisation)
8. [Coverage](#8-coverage)
9. [Worked Example](#9-worked-example)

---

## 1. Where to Start

If you are new to testing in Python, the full landscape — unit tests, integration tests, E2E tests, property-based tests — can feel daunting. You do not need all of it at once.

**Start here, in this order:**

### Step 1 — Install pytest and write your first test

```bash
uv add --dev pytest
```

Create a file called `tests/test_<your_module>.py` and write one test for the most important function in your code:

```python
from myapp.pricing import calculate_discount

def test_gold_customer_gets_ten_percent():
    assert calculate_discount(100.0, tier="gold") == 10.0
```

Run it:

```bash
uv run pytest tests/ -v
```

That is all you need to get started. One test is infinitely better than none.

### Step 2 — Cover your core logic with unit tests

Write unit tests (see [Section 2](#2-unit-tests)) for the functions and classes that contain your business logic. Aim for the scenarios that matter most: the happy path, the main error cases, and any edge cases you already know about. Don't aim for 100% coverage immediately — focus on the code that would be most painful to break.

### Step 3 — Add a smoke integration test

Once your unit tests are in good shape, add one or two integration tests (see [Section 3](#3-integration-tests)) that verify the most critical end-to-end flow against a real database or service. Even a single integration test that exercises the full stack gives you confidence that the pieces connect correctly.

### Step 4 — Add the rest as you need it

The remaining sections — E2E tests, property-based tests, mocking, coverage enforcement — are tools you reach for when a specific problem arises, not things to set up on day one. Return to them when:

- You break a workflow you thought was working → add an **E2E test**
- You discover an edge-case bug from unexpected input → add a **property-based test**
- A test is slow or flaky because of an external service → use **mocking**
- You want to prevent coverage from regressing in CI → set a **coverage threshold**

> **Rule of thumb:** Unit tests first, integration tests second, everything else when you have a reason.

---

## 2. Unit Tests

Unit tests verify a single function or class in isolation, with all external dependencies replaced by stubs or mocks.

### Install and run pytest

```bash
uv add --dev pytest
uv run pytest tests/ -v
```

### Write one assertion per logical concern

Each test should have a clear, single purpose. A failing test should tell you immediately *what* broke.

```python
# tests/test_pricing.py
import pytest
from myapp.pricing import calculate_discount

class TestCalculateDiscount:
    def test_no_discount_for_regular_customer(self):
        assert calculate_discount(100.0, tier="regular") == 0.0

    def test_ten_percent_discount_for_gold_customer(self):
        assert calculate_discount(100.0, tier="gold") == 10.0

    def test_unknown_tier_raises_value_error(self):
        with pytest.raises(ValueError, match="Unknown tier"):
            calculate_discount(100.0, tier="unknown")
```

### Use fixtures for shared setup and teardown

`@pytest.fixture` functions run setup code before each test and, when using `yield`, teardown code after.

```python
import pytest
from myapp.db import Database

@pytest.fixture
def db():
    database = Database(":memory:")
    database.migrate()
    yield database          # test runs here
    database.close()        # teardown always runs

def test_insert_user(db):
    db.insert_user(name="Alice")
    assert db.count_users() == 1

def test_empty_on_start(db):
    assert db.count_users() == 0
```

**Fixture scopes** control how often the fixture is created:

| Scope | Created once per… | Use when… |
|-------|-------------------|-----------|
| `function` *(default)* | test function | fixture has mutable state |
| `class` | test class | shared setup within a class |
| `module` | test file | expensive but read-only setup |
| `session` | entire test run | very expensive setup (e.g. one-time DB creation) |

```python
@pytest.fixture(scope="session")
def app_config():
    """Loaded once for the whole test session."""
    return load_config("tests/test_config.toml")
```

### Parametrize to eliminate duplication

`@pytest.mark.parametrize` runs the same test body with different inputs — far cleaner than copy-pasting tests.

```python
@pytest.mark.parametrize("amount,tier,expected", [
    (100.0, "regular", 0.0),
    (100.0, "gold",    10.0),
    (200.0, "gold",    20.0),
    (100.0, "platinum", 25.0),
])
def test_discount_table(amount, tier, expected):
    assert calculate_discount(amount, tier=tier) == expected
```

> **Tip:** Use `pytest.param(..., id="description")` to give parametrize cases human-readable names in the test output.

---

## 3. Integration Tests

Integration tests verify that multiple components work correctly together — typically involving real databases, queues, file systems, or network calls.

### Use real dependencies in a controlled environment

Spin up lightweight real dependencies with Docker (via `pytest-docker` or `testcontainers-python`) rather than mocking them. This catches issues that mocks can hide.

```bash
uv add --dev testcontainers
```

```python
import pytest
from testcontainers.postgres import PostgresContainer
from myapp.db import Database

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg

@pytest.fixture
def db(postgres):
    database = Database(postgres.get_connection_url())
    database.migrate()
    yield database
    database.rollback()   # reset state between tests

def test_create_and_fetch_user(db):
    db.insert_user(name="Bob", email="bob@example.com")
    user = db.get_user_by_email("bob@example.com")
    assert user.name == "Bob"
```

### Separate integration tests from unit tests

Mark integration tests explicitly so developers can run fast unit tests locally and reserve integration tests for CI.

```python
# tests/integration/test_db.py
import pytest

@pytest.mark.integration
def test_create_and_fetch_user(db):
    ...
```

Run selectively:

```bash
uv run pytest tests/ -m "not integration"   # fast: unit tests only
uv run pytest tests/ -m integration          # CI: integration tests only
uv run pytest tests/                         # everything
```

---

## 4. End-to-End Tests

End-to-end (E2E) tests exercise a complete workflow from the user's perspective — starting the application and interacting with it as a real user would.

### HTTP API testing with `httpx`

```bash
uv add --dev httpx pytest-anyio
```

```python
import pytest
import httpx
from myapp.main import create_app

@pytest.fixture
async def client():
    app = create_app()
    async with httpx.AsyncClient(app=app, base_url="http://test") as c:
        yield c

@pytest.mark.anyio
async def test_create_order_end_to_end(client):
    # Create a user
    resp = await client.post("/users", json={"name": "Alice"})
    assert resp.status_code == 201
    user_id = resp.json()["id"]

    # Place an order
    resp = await client.post("/orders", json={"user_id": user_id, "item": "Widget"})
    assert resp.status_code == 201
    order_id = resp.json()["id"]

    # Verify it appears in the list
    resp = await client.get(f"/users/{user_id}/orders")
    assert resp.status_code == 200
    order_ids = [o["id"] for o in resp.json()]
    assert order_id in order_ids
```

### CLI testing with `subprocess`

```python
import subprocess
import sys

def test_cli_version():
    result = subprocess.run(
        [sys.executable, "-m", "myapp", "--version"],
        capture_output=True, text=True,
    )
    assert result.returncode == 0
    assert "1." in result.stdout
```

> **Tip:** Keep E2E tests small in number. They are slow and brittle by nature. Use them only to verify the most critical user journeys; rely on unit and integration tests for thorough coverage.

---

## 5. Property-Based Testing

Property-based tests generate hundreds of random inputs automatically and check that a *property* (an invariant) holds for all of them. They are excellent for finding edge cases you wouldn't think to write by hand.

```bash
uv add --dev hypothesis
```

### A simple example

```python
from hypothesis import given, settings
from hypothesis import strategies as st
from myapp.encoding import encode, decode

@given(st.text())
def test_encode_decode_roundtrip(s):
    """Encoding then decoding always recovers the original string."""
    assert decode(encode(s)) == s
```

`hypothesis` will try thousands of inputs, including empty strings, Unicode edge cases, and very long strings — and if it finds a failure, it *shrinks* the input to the smallest example that still fails.

### Useful strategies

| Strategy | Generates |
|----------|-----------|
| `st.integers(min_value=0)` | Non-negative integers |
| `st.text()` | Unicode strings |
| `st.lists(st.integers())` | Lists of integers |
| `st.dictionaries(st.text(), st.integers())` | Dict with text keys |
| `st.one_of(st.none(), st.integers())` | `None` or an integer |
| `st.builds(MyClass, field=st.integers())` | Instances of a dataclass/class |

```python
from hypothesis import given
from hypothesis import strategies as st
from myapp.pricing import calculate_discount

@given(
    amount=st.floats(min_value=0.01, max_value=1_000_000, allow_nan=False),
    tier=st.sampled_from(["regular", "gold", "platinum"]),
)
def test_discount_never_exceeds_amount(amount, tier):
    discount = calculate_discount(amount, tier=tier)
    assert 0 <= discount <= amount
```

> **Tip:** Hypothesis stores failing examples in a local database (`~/.hypothesis/`) and replays them on subsequent runs, so a flaky edge case is never silently forgotten.

---

## 6. Mocking & Patching

Use mocks to replace real dependencies (HTTP clients, email services, third-party APIs) during unit tests, keeping tests fast and deterministic.

### `unittest.mock` — standard library

```python
from unittest.mock import MagicMock, patch
from myapp.notifications import send_welcome_email

def test_send_welcome_email_calls_smtp():
    with patch("myapp.notifications.smtplib.SMTP") as mock_smtp:
        mock_instance = mock_smtp.return_value.__enter__.return_value
        send_welcome_email("alice@example.com")
        mock_instance.sendmail.assert_called_once()
        args = mock_instance.sendmail.call_args[0]
        assert "alice@example.com" in args
```

### `pytest-mock` — cleaner fixture-based API

`pytest-mock` wraps `unittest.mock` in a `mocker` fixture that automatically cleans up patches after each test — no context managers needed.

```bash
uv add --dev pytest-mock
```

```python
def test_send_welcome_email_calls_smtp(mocker):
    mock_smtp = mocker.patch("myapp.notifications.smtplib.SMTP")
    mock_instance = mock_smtp.return_value.__enter__.return_value

    send_welcome_email("alice@example.com")

    mock_instance.sendmail.assert_called_once()
```

### When *not* to mock

Mocking the wrong thing creates tests that pass even when the real code is broken. Avoid mocking:

- **Your own code** — test it for real instead.
- **Anything that's already fast and deterministic** — an in-memory SQLite DB is better than a mocked one.
- **The thing you're testing** — if you mock the function under test, you're not testing anything.

Prefer mocking at the **boundary of your system**: external HTTP calls, the system clock (`datetime.now`), email/SMS gateways, and file I/O on large or remote files.

```python
# Good: mock the external HTTP call, test the business logic
def test_get_exchange_rate_retries_on_failure(mocker):
    mock_get = mocker.patch("myapp.forex.httpx.get")
    mock_get.side_effect = [httpx.TimeoutException(""), httpx.Response(200, json={"rate": 1.25})]

    rate = get_exchange_rate("USD", "GBP")

    assert rate == 1.25
    assert mock_get.call_count == 2
```

---

## 7. Test Organisation

### Directory layout

Mirror the `src/` layout with a sibling `tests/` directory:

```
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── orders.py
│       └── pricing.py
├── tests/
│   ├── conftest.py          # shared fixtures, plugins
│   ├── unit/
│   │   ├── test_orders.py
│   │   └── test_pricing.py
│   └── integration/
│       └── test_db.py
├── pyproject.toml
└── README.md
```

### `conftest.py`

`conftest.py` files are automatically discovered by pytest. Use them to:

- Define shared fixtures available to all tests in the same directory and below.
- Register custom markers.
- Configure plugins.

```python
# tests/conftest.py
import pytest
from myapp.db import Database

def pytest_configure(config):
    config.addinivalue_line("markers", "integration: mark test as requiring a live DB")
    config.addinivalue_line("markers", "slow: mark test as slow-running")

@pytest.fixture(scope="session")
def app_config():
    return {"env": "test", "debug": True}
```

### `pyproject.toml` configuration

Centralise pytest settings to avoid repeating CLI flags:

```toml
[tool.pytest.ini_options]
testpaths   = ["tests"]
addopts     = "-v --tb=short"
markers     = [
    "integration: requires live external services",
    "slow: takes more than a few seconds",
]
```

### Naming conventions

- Test files: `test_<module>.py`
- Test functions: `test_<what>_<condition>` — e.g. `test_checkout_raises_on_empty_cart`
- Test classes: `Test<Subject>` — e.g. `TestCheckout`
- Fixtures: noun phrases — e.g. `db`, `authenticated_client`, `sample_order`

---

## 8. Coverage

Coverage measures which lines of source code are executed during your test run.

```bash
uv add --dev pytest-cov
uv run pytest tests/ --cov=src/myapp --cov-report=term-missing
```

```
Name                        Stmts   Miss  Cover   Missing
---------------------------------------------------------
src/myapp/orders.py            42      3    93%   87, 102-103
src/myapp/pricing.py           28      0   100%
---------------------------------------------------------
TOTAL                          70      3    96%
```

### Set a minimum threshold in CI

```toml
[tool.pytest.ini_options]
addopts = "--cov=src/myapp --cov-fail-under=80"
```

The test run will exit non-zero if coverage drops below 80%, catching regressions automatically.

### What to aim for

- Target **80–90%** as a practical baseline for application code.
- 100% coverage does not mean 100% correctness — focus on meaningful assertions, not just line hits.
- Use `# pragma: no cover` sparingly for genuinely untestable branches (e.g. platform-specific fallbacks, `if __name__ == "__main__"` guards).

```python
if sys.platform == "win32":  # pragma: no cover
    ...
```

### Generate HTML reports for local exploration

```bash
uv run pytest --cov=src/myapp --cov-report=html
open htmlcov/index.html
```

---

## 9. Worked Example

A complete, minimal project showing how the pieces fit together.

```
my_project/
├── src/myapp/
│   ├── __init__.py
│   └── cart.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   └── test_cart.py
│   └── integration/
│       └── test_cart_db.py
└── pyproject.toml
```

```python
# src/myapp/cart.py
from __future__ import annotations
from dataclasses import dataclass, field


class CartError(Exception):
    """Base exception for cart operations."""


@dataclass
class Cart:
    items: list[dict] = field(default_factory=list)

    def add(self, name: str, price: float, qty: int = 1) -> None:
        if price < 0:
            raise CartError(f"Price cannot be negative: {price}")
        self.items.append({"name": name, "price": price, "qty": qty})

    def total(self) -> float:
        return sum(i["price"] * i["qty"] for i in self.items)

    def is_empty(self) -> bool:
        return len(self.items) == 0
```

```python
# tests/conftest.py
import pytest
from myapp.cart import Cart

def pytest_configure(config):
    config.addinivalue_line("markers", "integration: requires a live database")

@pytest.fixture
def empty_cart() -> Cart:
    return Cart()

@pytest.fixture
def stocked_cart() -> Cart:
    cart = Cart()
    cart.add("Widget", price=9.99, qty=2)
    cart.add("Gadget", price=24.99)
    return cart
```

```python
# tests/unit/test_cart.py
import pytest
from hypothesis import given
from hypothesis import strategies as st
from myapp.cart import Cart, CartError


class TestCartAdd:
    def test_add_single_item(self, empty_cart):
        empty_cart.add("Widget", price=9.99)
        assert len(empty_cart.items) == 1

    def test_negative_price_raises(self, empty_cart):
        with pytest.raises(CartError, match="negative"):
            empty_cart.add("Widget", price=-1.0)

    def test_is_empty_initially(self, empty_cart):
        assert empty_cart.is_empty()

    def test_not_empty_after_add(self, stocked_cart):
        assert not stocked_cart.is_empty()


class TestCartTotal:
    @pytest.mark.parametrize("items,expected", [
        ([], 0.0),
        ([("A", 10.0, 1)], 10.0),
        ([("A", 10.0, 3), ("B", 5.0, 2)], 40.0),
    ])
    def test_total(self, items, expected):
        cart = Cart()
        for name, price, qty in items:
            cart.add(name, price=price, qty=qty)
        assert cart.total() == pytest.approx(expected)

    @given(
        prices=st.lists(st.floats(min_value=0.0, max_value=1000.0, allow_nan=False), min_size=1),
    )
    def test_total_never_negative(self, prices):
        cart = Cart()
        for i, price in enumerate(prices):
            cart.add(f"item_{i}", price=price)
        assert cart.total() >= 0
```

```python
# tests/integration/test_cart_db.py
import pytest
from myapp.cart import Cart

@pytest.mark.integration
def test_saved_cart_can_be_reloaded(db):
    cart = Cart()
    cart.add("Widget", price=9.99)
    cart_id = db.save_cart(cart)

    loaded = db.load_cart(cart_id)
    assert loaded.total() == pytest.approx(9.99)
```

```toml
# pyproject.toml (test section)
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts   = "-v --tb=short --cov=src/myapp --cov-report=term-missing --cov-fail-under=80"
markers   = [
    "integration: requires a live database",
]
```

---

## Further Reading

- [pytest documentation](https://docs.pytest.org/)
- [Hypothesis documentation](https://hypothesis.readthedocs.io/)
- [pytest-mock](https://pytest-mock.readthedocs.io/)
- [testcontainers-python](https://testcontainers-community.github.io/testcontainers-python/)
- [Coverage.py documentation](https://coverage.readthedocs.io/)
- [Google Testing Blog](https://testing.googleblog.com/)
