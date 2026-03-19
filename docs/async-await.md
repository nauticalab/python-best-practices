# Python async/await Best Practices

A practical guide to writing correct, efficient, and maintainable asynchronous Python code with `asyncio`.

---

## Table of Contents

1. [When to Use async/await](#1-when-to-use-asyncawait)
2. [asyncio Basics](#2-asyncio-basics)
3. [Tasks and Concurrency](#3-tasks-and-concurrency)
4. [Common Patterns](#4-common-patterns)
5. [Error Handling](#5-error-handling)
6. [Testing Async Code](#6-testing-async-code)
7. [Worked Example](#7-worked-example)
8. [Common Anti-Patterns](#8-common-anti-patterns)

---

## 1. When to Use async/await

### The right problem: I/O-bound concurrency

`async/await` solves one specific problem well: waiting on many I/O operations — network requests, database queries, file reads — without tying up an OS thread for each one. The event loop interleaves thousands of concurrent "waits" in a single thread.

```python
# Without async: each request blocks the thread — requests run serially
import httpx

def fetch_all(urls: list[str]) -> list[str]:
    results = []
    for url in urls:
        results.append(httpx.get(url).text)   # blocks here until response arrives
    return results

# With async: all requests are in-flight simultaneously
import asyncio
import httpx

async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.text for r in responses]
```

### The wrong problem: CPU-bound work

`async/await` does **not** speed up CPU-bound code. Because of the GIL, Python can only execute one thread at a time, and the async event loop runs entirely in one thread. Awaiting a CPU-heavy coroutine just hands control back to the event loop after blocking it.

| Workload | Right tool |
|----------|-----------|
| Many concurrent HTTP calls | `asyncio` + async HTTP client |
| Database queries (many concurrent) | `asyncio` + async DB driver |
| Streaming file/socket I/O | `asyncio` |
| CPU-heavy computation (image processing, ML inference) | `concurrent.futures.ProcessPoolExecutor` / `multiprocessing` |
| Parallelising a handful of blocking calls | `concurrent.futures.ThreadPoolExecutor` |

Use `asyncio.to_thread` (Python 3.9+) or `loop.run_in_executor()` to run blocking or CPU-bound code from an async context without blocking the event loop.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

async def main() -> None:
    loop = asyncio.get_event_loop()
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10_000_000)
    print(result)

# Simpler for blocking I/O in threads (Python 3.9+)
async def read_file(path: str) -> str:
    return await asyncio.to_thread(open(path).read)
```

> **Rule of thumb:** If your code mostly waits on the network or a database, `asyncio` will help. If it mostly computes, reach for `multiprocessing` instead.

---

## 2. asyncio Basics

### Coroutines, awaitables, and the event loop

A **coroutine** is an `async def` function. Calling it returns a coroutine object — it does not execute until it is awaited or scheduled as a task.

```python
import asyncio

async def greet(name: str) -> str:
    await asyncio.sleep(0.1)   # yields control to the event loop
    return f"Hello, {name}!"

# greet("Alice")  ← returns a coroutine object; nothing runs yet

result = asyncio.run(greet("Alice"))   # starts the event loop and runs to completion
print(result)   # Hello, Alice!
```

`asyncio.run()` is the canonical entry point for async programs (Python 3.7+). It:
- Creates a new event loop
- Runs the coroutine to completion
- Closes the loop and cancels any lingering tasks

Never call `asyncio.run()` from inside a running event loop (e.g. from within a coroutine). If you're already in async context, just `await` the coroutine directly.

### The `await` expression

`await` can be used on any **awaitable**: a coroutine, a `Task`, a `Future`, or any object with an `__await__` method. It suspends the current coroutine, yields control to the event loop, and resumes when the awaitable completes.

```python
async def pipeline() -> None:
    # Each await suspends this coroutine until the operation completes.
    # Other tasks can run during each suspension.
    response = await fetch("https://api.example.com/data")
    parsed   = await parse(response)
    await store(parsed)
```

### `async with` and `async for`

Many async libraries provide async context managers and async iterables:

```python
import asyncio
import aiofiles

# Async context manager
async def write_file(path: str, content: str) -> None:
    async with aiofiles.open(path, "w") as f:
        await f.write(content)

# Async iterator
async def stream_lines(reader: asyncio.StreamReader) -> None:
    async for line in reader:
        process(line)
```

---

## 3. Tasks and Concurrency

Running coroutines sequentially with `await` is fine when they depend on each other. For independent operations, wrap them in **Tasks** so they run concurrently.

### `asyncio.create_task`

`create_task` schedules a coroutine to run as a background task. The task starts immediately (on the next event loop iteration) and runs concurrently with the current coroutine.

```python
import asyncio

async def slow_op(label: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return label

async def main() -> None:
    # Sequential: total time ≈ 1.0 + 0.5 = 1.5 s
    a = await slow_op("a", 1.0)
    b = await slow_op("b", 0.5)

    # Concurrent: total time ≈ max(1.0, 0.5) = 1.0 s
    task_a = asyncio.create_task(slow_op("a", 1.0))
    task_b = asyncio.create_task(slow_op("b", 0.5))
    a = await task_a
    b = await task_b
```

> **Important:** Always keep a reference to created tasks. If you don't, the garbage collector may destroy the task before it completes. Assign it to a variable or store it in a set.

```python
# Common pattern: fire-and-forget with cleanup reference
_background_tasks: set[asyncio.Task] = set()

def fire_and_forget(coro) -> asyncio.Task:
    task = asyncio.create_task(coro)
    _background_tasks.add(task)
    task.add_done_callback(_background_tasks.discard)
    return task
```

### `asyncio.gather`

`gather` runs multiple awaitables concurrently and collects their results in order. It's the most common way to fan out across a list of tasks.

```python
import asyncio
import httpx

async def fetch(client: httpx.AsyncClient, url: str) -> str:
    response = await client.get(url)
    response.raise_for_status()
    return response.text

async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*[fetch(client, url) for url in urls])
```

By default, if any coroutine raises, `gather` immediately propagates the exception and cancels the rest. Pass `return_exceptions=True` to collect exceptions as results instead:

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
for result in results:
    if isinstance(result, Exception):
        logger.error("Task failed: %s", result)
    else:
        process(result)
```

### `asyncio.wait`

`wait` gives more granular control — you specify a condition (`FIRST_COMPLETED`, `FIRST_EXCEPTION`, or `ALL_COMPLETED`) and get back two sets: done and pending.

```python
import asyncio

async def race(tasks: list[asyncio.Task]) -> object:
    """Return the result of whichever task finishes first."""
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    for task in pending:
        task.cancel()
    return done.pop().result()
```

### `asyncio.TaskGroup` (Python 3.11+)

`TaskGroup` is the preferred modern API for structured concurrency. It ensures all tasks are either completed or cancelled when the block exits — preventing tasks from being silently abandoned.

```python
import asyncio

async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        task_a = tg.create_task(slow_op("a", 1.0))
        task_b = tg.create_task(slow_op("b", 0.5))
    # Both tasks are guaranteed to be done here (or an exception was raised)
    print(task_a.result(), task_b.result())
```

If any task raises, the group cancels all remaining tasks and re-raises the exception. If multiple tasks raise, they are collected into an `ExceptionGroup`.

> **Recommendation:** Prefer `TaskGroup` for new Python 3.11+ code. Use `gather` for compatibility with earlier versions.

### Cancellation

Tasks can be cancelled explicitly. A `CancelledError` is raised at the next `await` point inside the task.

```python
import asyncio

async def long_running() -> None:
    try:
        await asyncio.sleep(60)
    except asyncio.CancelledError:
        # Perform cleanup here (close connections, flush buffers, etc.)
        print("Task was cancelled — cleaning up")
        raise   # Always re-raise CancelledError

async def main() -> None:
    task = asyncio.create_task(long_running())
    await asyncio.sleep(1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Task confirmed cancelled")
```

> **Always re-raise `CancelledError`** after cleanup. Swallowing it prevents cooperative cancellation from propagating correctly.

---

## 4. Common Patterns

### Timeouts

Use `asyncio.timeout` (Python 3.11+) or `asyncio.wait_for` to enforce deadlines:

```python
import asyncio

# Python 3.11+: asyncio.timeout context manager
async def fetch_with_timeout(url: str) -> str:
    try:
        async with asyncio.timeout(5.0):
            return await fetch(url)
    except TimeoutError:
        logger.warning("Request to %s timed out", url)
        raise

# Python 3.8+: asyncio.wait_for
async def fetch_with_timeout_compat(url: str) -> str:
    try:
        return await asyncio.wait_for(fetch(url), timeout=5.0)
    except asyncio.TimeoutError:
        logger.warning("Request to %s timed out", url)
        raise
```

### Limiting concurrency with `asyncio.Semaphore`

Without rate limiting, `gather` over thousands of URLs will try to open thousands of connections at once. Use a semaphore to cap concurrency:

```python
import asyncio
import httpx

async def fetch_limited(
    client: httpx.AsyncClient,
    url: str,
    sem: asyncio.Semaphore,
) -> str:
    async with sem:   # at most N coroutines inside here at once
        response = await client.get(url)
        return response.text

async def fetch_all(urls: list[str], concurrency: int = 20) -> list[str]:
    sem = asyncio.Semaphore(concurrency)
    async with httpx.AsyncClient() as client:
        tasks = [fetch_limited(client, url, sem) for url in urls]
        return await asyncio.gather(*tasks)
```

### Producer/consumer with `asyncio.Queue`

`Queue` decouples producers from consumers and naturally handles backpressure:

```python
import asyncio

async def producer(queue: asyncio.Queue[str], items: list[str]) -> None:
    for item in items:
        await queue.put(item)
    await queue.put(None)   # sentinel to signal completion

async def consumer(queue: asyncio.Queue[str]) -> list[str]:
    results = []
    while True:
        item = await queue.get()
        if item is None:
            break
        results.append(await process(item))
        queue.task_done()
    return results

async def pipeline(items: list[str]) -> list[str]:
    queue: asyncio.Queue[str] = asyncio.Queue(maxsize=10)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue, items))
        consumer_task = tg.create_task(consumer(queue))
    return consumer_task.result()
```

### Async context managers

Implement `__aenter__` and `__aexit__` for resources that need async setup/teardown, or use `contextlib.asynccontextmanager` for a generator-based shorthand:

```python
from contextlib import asynccontextmanager
import asyncio

@asynccontextmanager
async def managed_connection(host: str, port: int):
    reader, writer = await asyncio.open_connection(host, port)
    try:
        yield reader, writer
    finally:
        writer.close()
        await writer.wait_closed()

async def main() -> None:
    async with managed_connection("localhost", 8080) as (reader, writer):
        writer.write(b"hello\n")
        response = await reader.readline()
```

---

## 5. Error Handling

### Exceptions propagate from `await`

Exceptions raised inside an awaited coroutine propagate naturally — treat them like synchronous exceptions:

```python
import asyncio
import httpx

async def safe_fetch(url: str) -> str | None:
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            response.raise_for_status()
            return response.text
    except httpx.HTTPStatusError as exc:
        logger.error("HTTP %d for %s", exc.response.status_code, url)
        return None
    except httpx.RequestError as exc:
        logger.error("Network error fetching %s: %s", url, exc)
        return None
```

### Exceptions in `gather`

By default, `gather` cancels remaining tasks and raises on the first exception. Use `return_exceptions=True` when you want to inspect all outcomes:

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
successes = [r for r in results if not isinstance(r, BaseException)]
failures  = [r for r in results if isinstance(r, BaseException)]
```

### `ExceptionGroup` with `TaskGroup` (Python 3.11+)

When multiple tasks in a `TaskGroup` raise, Python 3.11+ wraps them in an `ExceptionGroup`. Use `except*` to handle specific exception types within the group:

```python
import asyncio

async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(might_raise_value_error())
            tg.create_task(might_raise_type_error())
    except* ValueError as eg:
        for exc in eg.exceptions:
            logger.error("ValueError: %s", exc)
    except* TypeError as eg:
        for exc in eg.exceptions:
            logger.error("TypeError: %s", exc)
```

### Cleaning up with `finally` and `asyncio.shield`

`finally` blocks run even when a coroutine is cancelled, making them the right place for cleanup:

```python
async def with_cleanup() -> None:
    resource = await acquire_resource()
    try:
        await do_work(resource)
    finally:
        await resource.release()   # always runs, even on cancellation
```

Use `asyncio.shield` to protect a cleanup coroutine from being cancelled mid-flight:

```python
async def safe_cleanup(resource) -> None:
    # Even if the outer task is cancelled, flush_to_disk will complete
    await asyncio.shield(resource.flush_to_disk())
```

### Handling unhandled task exceptions

By default, unhandled exceptions in discarded tasks are logged by asyncio but not re-raised. Set a custom exception handler to surface them prominently:

```python
import asyncio
import logging

logger = logging.getLogger(__name__)

def handle_task_exception(loop: asyncio.AbstractEventLoop, context: dict) -> None:
    exc = context.get("exception")
    if exc:
        logger.critical("Unhandled exception in task: %s", exc, exc_info=exc)
    else:
        logger.critical("Unhandled asyncio error: %s", context["message"])

asyncio.get_event_loop().set_exception_handler(handle_task_exception)
```

---

## 6. Testing Async Code

### Set up `pytest-asyncio`

```bash
uv add --dev pytest pytest-asyncio
```

Configure it in `pyproject.toml` to avoid decorating every test manually:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"   # all async test functions are treated as async tests
```

### Write async tests

```python
# tests/test_fetcher.py
import pytest
import respx   # uv add --dev respx

from myapp.fetcher import fetch_all

@pytest.mark.asyncio   # not needed with asyncio_mode = "auto"
async def test_fetch_all_returns_bodies():
    urls = ["https://example.com/a", "https://example.com/b"]

    with respx.mock:
        respx.get("https://example.com/a").respond(text="body-a")
        respx.get("https://example.com/b").respond(text="body-b")
        results = await fetch_all(urls)

    assert results == ["body-a", "body-b"]

async def test_fetch_all_handles_http_error():
    with respx.mock:
        respx.get("https://example.com/bad").respond(status_code=500)
        results = await fetch_all(["https://example.com/bad"])

    assert results == [None]
```

### Async fixtures

```python
import pytest

@pytest.fixture
async def db():
    """Async fixture: yields a connected in-memory database."""
    from myapp.db import Database
    database = await Database.connect(":memory:")
    await database.migrate()
    yield database
    await database.close()

async def test_insert_user(db):
    await db.insert_user(name="Alice")
    assert await db.count_users() == 1
```

### Mocking async dependencies

Use `unittest.mock.AsyncMock` (Python 3.8+) for mocking coroutines:

```python
from unittest.mock import AsyncMock, patch
import myapp.service

async def test_calls_external_service():
    mock_response = AsyncMock(return_value={"status": "ok"})

    with patch("myapp.service.call_api", mock_response):
        result = await myapp.service.process("input")

    mock_response.assert_awaited_once_with("input")
    assert result == "ok"
```

> **Tip:** `AsyncMock` automatically handles `await`, so `await mock_fn()` works correctly. Regular `MagicMock` will not — attempting to `await` a `MagicMock` raises a `TypeError`.

---

## 7. Worked Example

A small async HTTP scraper that demonstrates concurrency limiting, error handling, timeouts, and structured task management.

```python
# myapp/scraper.py
"""Async URL scraper with concurrency control and timeout handling."""
from __future__ import annotations

import asyncio
import logging
from dataclasses import dataclass

import httpx

logger = logging.getLogger(__name__)

DEFAULT_CONCURRENCY = 20
DEFAULT_TIMEOUT = 10.0


@dataclass
class ScrapeResult:
    url: str
    body: str | None = None
    error: str | None = None

    @property
    def ok(self) -> bool:
        return self.error is None


async def _fetch_one(
    client: httpx.AsyncClient,
    url: str,
    sem: asyncio.Semaphore,
    timeout: float,
) -> ScrapeResult:
    async with sem:
        try:
            async with asyncio.timeout(timeout):
                response = await client.get(url)
                response.raise_for_status()
                return ScrapeResult(url=url, body=response.text)
        except TimeoutError:
            logger.warning("Timeout fetching %s", url)
            return ScrapeResult(url=url, error="timeout")
        except httpx.HTTPStatusError as exc:
            logger.warning("HTTP %d for %s", exc.response.status_code, url)
            return ScrapeResult(url=url, error=f"http_{exc.response.status_code}")
        except httpx.RequestError as exc:
            logger.warning("Network error for %s: %s", url, exc)
            return ScrapeResult(url=url, error="network_error")


async def scrape(
    urls: list[str],
    *,
    concurrency: int = DEFAULT_CONCURRENCY,
    timeout: float = DEFAULT_TIMEOUT,
) -> list[ScrapeResult]:
    """Fetch all URLs concurrently, respecting the concurrency limit.

    Args:
        urls: List of URLs to fetch.
        concurrency: Maximum number of simultaneous in-flight requests.
        timeout: Per-request timeout in seconds.

    Returns:
        List of ScrapeResult objects in the same order as ``urls``.
    """
    sem = asyncio.Semaphore(concurrency)
    async with httpx.AsyncClient(follow_redirects=True) as client:
        tasks = [_fetch_one(client, url, sem, timeout) for url in urls]
        return await asyncio.gather(*tasks)


# --- entry point ---

async def main() -> None:
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")

    urls = [
        "https://httpbin.org/get",
        "https://httpbin.org/status/404",
        "https://httpbin.org/delay/15",   # will time out
    ]

    results = await scrape(urls, concurrency=5, timeout=5.0)

    ok = [r for r in results if r.ok]
    failed = [r for r in results if not r.ok]
    logger.info("Scraped %d URLs: %d ok, %d failed", len(results), len(ok), len(failed))
    for r in failed:
        logger.warning("  FAILED %s — %s", r.url, r.error)


if __name__ == "__main__":
    asyncio.run(main())
```

```python
# tests/test_scraper.py
import respx
from myapp.scraper import scrape

async def test_scrape_success():
    with respx.mock:
        respx.get("https://example.com/a").respond(text="hello")
        respx.get("https://example.com/b").respond(text="world")
        results = await scrape(["https://example.com/a", "https://example.com/b"])

    assert [r.body for r in results] == ["hello", "world"]
    assert all(r.ok for r in results)

async def test_scrape_handles_http_error():
    with respx.mock:
        respx.get("https://example.com/bad").respond(status_code=500)
        results = await scrape(["https://example.com/bad"])

    assert results[0].ok is False
    assert results[0].error == "http_500"

async def test_scrape_preserves_order():
    urls = [f"https://example.com/{i}" for i in range(5)]
    with respx.mock:
        for i, url in enumerate(urls):
            respx.get(url).respond(text=str(i))
        results = await scrape(urls)

    assert [r.body for r in results] == ["0", "1", "2", "3", "4"]
```

---

## 8. Common Anti-Patterns

### Blocking the event loop

Calling any blocking function — `time.sleep`, `requests.get`, CPU-heavy code — directly inside a coroutine freezes the entire event loop for all other tasks.

```python
import time
import asyncio

# Bad: blocks the event loop — no other tasks can run during the sleep
async def bad_delay() -> None:
    time.sleep(5)   # freezes everything

# Good: yields control to the event loop
async def good_delay() -> None:
    await asyncio.sleep(5)

# Good: run blocking code in a thread pool
async def good_blocking_io(path: str) -> str:
    return await asyncio.to_thread(open(path).read)
```

### Calling `asyncio.get_event_loop()` to run coroutines

`asyncio.get_event_loop()` is a legacy API with subtle pitfalls. Use `asyncio.run()` at the entry point and `asyncio.get_running_loop()` if you need the loop from within async code.

```python
# Avoid — deprecated pattern, breaks with Python 3.10+ in some contexts
loop = asyncio.get_event_loop()
loop.run_until_complete(main())

# Correct — always use asyncio.run() at the top level
asyncio.run(main())

# Correct — use get_running_loop() inside async code
async def schedule_callback() -> None:
    loop = asyncio.get_running_loop()
    loop.call_later(1.0, some_sync_callback)
```

### Forgetting to `await` a coroutine

Forgetting `await` is a silent bug — the coroutine object is created but never executed. Enable asyncio's debug mode or use a type checker (Pyright flags un-awaited coroutines as errors).

```python
async def save(data: dict) -> None:
    ...

async def process(data: dict) -> None:
    result = transform(data)
    save(result)       # Bug: coroutine created but never awaited!
    await save(result) # Correct
```

```bash
# Run with asyncio debug mode to catch un-awaited coroutines at runtime
PYTHONASYNCIODEBUG=1 python -m myapp
```

### Discarding tasks without a reference

As noted in [Section 3](#3-tasks-and-concurrency), tasks not held by a variable can be garbage-collected mid-execution.

```python
# Bug: task may be collected before it completes
asyncio.create_task(cleanup())

# Correct: keep a reference
task = asyncio.create_task(cleanup())
await task

# Correct for true fire-and-forget: use the _background_tasks pattern
fire_and_forget(cleanup())
```

### Wrapping sync code in a coroutine without yielding

Adding `async def` to a function that never `await`s anything gains nothing — it runs synchronously in the event loop and can still block it if it does heavy work.

```python
# Pointless — never yields to the event loop; same behaviour as a sync function
async def compute(n: int) -> int:
    return sum(i * i for i in range(n))   # still blocks the loop

# Correct for CPU work: run in an executor
async def compute(n: int) -> int:
    return await asyncio.to_thread(lambda: sum(i * i for i in range(n)))
```

### Mixing `asyncio.run()` with a running loop

`asyncio.run()` creates a *new* event loop and refuses to run inside an existing one. In environments like Jupyter notebooks where a loop is already running, use `nest_asyncio` or simply `await` the coroutine directly.

```python
# Raises RuntimeError("This event loop is already running")
# when called from inside a running loop (e.g. Jupyter, FastAPI startup)
asyncio.run(my_coroutine())

# In a Jupyter notebook (or other running-loop context), just await:
result = await my_coroutine()

# Or install nest_asyncio as a last resort:
import nest_asyncio
nest_asyncio.apply()
asyncio.run(my_coroutine())
```

---

## Further Reading

- [Python Docs — asyncio](https://docs.python.org/3/library/asyncio.html)
- [Python Docs — Developing with asyncio](https://docs.python.org/3/library/asyncio-dev.html)
- [PEP 492 — Coroutines with async and await syntax](https://peps.python.org/pep-0492/)
- [PEP 654 — Exception Groups and except*](https://peps.python.org/pep-0654/)
- [httpx — Async HTTP client](https://www.python-httpx.org/)
- [pytest-asyncio documentation](https://pytest-asyncio.readthedocs.io/)
- [trio — An alternative async framework](https://trio.readthedocs.io/)
