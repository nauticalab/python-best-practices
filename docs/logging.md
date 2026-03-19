# Python Logging Best Practices

A concise guide to configuring and using Python's logging system effectively in production code.

---

## Table of Contents

1. [Use `logging`, Not `print()`](#1-use-logging-not-print)
2. [Name Loggers with `__name__`](#2-name-loggers-with-__name__)
3. [Log Levels](#3-log-levels)
4. [Writing Effective Log Messages](#4-writing-effective-log-messages)
5. [Configuration Patterns](#5-configuration-patterns)
6. [Structured Logging](#6-structured-logging)
7. [Worked Example](#7-worked-example)

---

## 1. Use `logging`, Not `print()`

`print()` feels natural — it's instant feedback, no setup required. For a quick script or a throwaway experiment, it's perfectly fine. But as soon as your code runs in production or gets used by others, `print()` starts to show its limits.

Here's why it falls short:

- **No severity levels.** Every `print()` always fires. To suppress debug noise in production you have to delete, comment out, or gate every call behind an `if DEBUG:` check — and remember to undo that before the next release.
- **No timestamps or context.** Plain `print()` output gives you the message and nothing else. When you're debugging an incident at 2 am, you really want to know *when* each line was logged and *which module* it came from.
- **Stdout only.** `print()` goes to one place. If you need errors in stderr, a rotating file, *and* a cloud log service, you're refactoring. The `logging` module routes the same log call to as many destinations as you like.
- **No runtime control.** Changing verbosity means editing code and redeploying. With `logging` you can flip the level via an environment variable or config file — no code change needed.
- **Thread-safety surprises.** In multi-threaded applications, concurrent `print()` calls can interleave mid-line, producing garbled output. `logging` handlers use locks to keep messages coherent.
- **Invisible to log aggregators.** Tools like Datadog, Loki, and Splunk are built around structured log records. Raw `print()` output is just an untagged string they can't easily parse or index.

Switching is straightforward — the API is almost as simple as `print()`:

```python
# Avoid in production code
print(f"Processing order {order_id}")
print(f"ERROR: payment failed for {order_id}")

# Prefer
import logging

logger = logging.getLogger(__name__)
logger.info("Processing order %s", order_id)
logger.error("Payment failed for order %s", order_id)
```

> **Tip:** Use `%`-style formatting (`logger.info("x=%s", x)`) rather than f-strings — the string is only interpolated if the message is actually emitted, saving CPU on suppressed log levels.

---

## 2. Name Loggers with `__name__`

Always create loggers using `logging.getLogger(__name__)`. This names the logger after the module's fully-qualified import path (e.g. `myapp.payments.processor`), which:

- Mirrors your package hierarchy, making it easy to filter or silence entire subsystems
- Avoids naming collisions across libraries
- Lets the root logger act as a catch-all without extra configuration

```python
# myapp/payments/processor.py
import logging

logger = logging.getLogger(__name__)
# logger.name == "myapp.payments.processor"
```

**Never** use a bare string like `logging.getLogger("my_logger")` — it creates a flat, global name with no hierarchy.

**Never** call `logging.basicConfig()` inside a library — doing so configures the root logger and surprises consumers of your package. Leave configuration to the application entry point.

---

## 3. Log Levels

Python's logging levels, from least to most severe:

| Level | Value | When to use |
|-------|-------|-------------|
| `DEBUG` | 10 | Detailed diagnostic info, useful only during development (`loop iteration i=42`, SQL queries) |
| `INFO` | 20 | Confirmation that things are working as expected (`server started on port 8080`, `job completed`) |
| `WARNING` | 30 | Something unexpected happened, but the program is still running (`deprecated API used`, `retry attempt 2/3`) |
| `ERROR` | 40 | A serious problem caused an operation to fail, but the process continues (`failed to save record`, `payment declined`) |
| `CRITICAL` | 50 | A severe error that will likely cause the process to abort (`database unreachable`, `out of disk space`) |

```python
logger.debug("Cache miss for key %s, querying DB", key)
logger.info("User %s authenticated successfully", user_id)
logger.warning("Config key 'timeout' missing, using default 30s")
logger.error("Failed to send email to %s: %s", recipient, exc, exc_info=True)
logger.critical("Cannot connect to primary DB — shutting down")
```

> **Rule of thumb:** In production, run at `INFO`. In development or when debugging, drop to `DEBUG`. Only escalate to `WARNING`/`ERROR`/`CRITICAL` for genuinely actionable conditions.

---

## 4. Writing Effective Log Messages

A log message is only useful if someone — or a search query — can act on it. These guidelines help you write messages that are easy to find, understand, and triage.

### Be specific and include context

Tell the reader *what* happened and *which* entity was involved. Vague messages force engineers to cross-reference code during an incident.

```python
# Hard to act on — what failed? which user?
logger.error("Request failed")
logger.info("Done")

# Actionable — includes the operation, identifier, and reason
logger.error("Failed to charge customer %s: %s", customer_id, exc)
logger.info("Order %s dispatched to warehouse %s", order_id, warehouse_id)
```

### Use consistent, grep-friendly verbs

Pick a small vocabulary of action verbs and use them consistently across the codebase. This makes `grep`/log-query filtering reliable.

```python
# Inconsistent — hard to search across services
logger.info("User login successful")
logger.info("Authenticated user")
logger.info("Login OK for %s", username)

# Consistent — one canonical verb per lifecycle event
logger.info("user.login succeeded user_id=%s", user_id)
logger.info("user.logout user_id=%s reason=%s", user_id, reason)
logger.info("user.created user_id=%s email=%s", user_id, email)
```

### State the outcome, not just the intent

Log *after* an operation completes (or fails), not only before it starts. Paired entry/exit logs make it easy to spot operations that started but never finished.

```python
logger.debug("Fetching config from %s", config_url)          # before
config = fetch_config(config_url)
logger.debug("Config fetched: %d keys loaded", len(config))  # after (outcome)
```

### Always log exceptions with `exc_info=True`

Pass `exc_info=True` (or use `logger.exception()`) so the full traceback is captured. A message alone rarely provides enough context to debug.

```python
try:
    result = call_external_api(payload)
except TimeoutError as exc:
    # logger.exception is shorthand for logger.error(..., exc_info=True)
    logger.exception("API call timed out after %ds payload_size=%d", timeout, len(payload))
    raise
```

### Never log sensitive data

Passwords, tokens, API keys, and PII must never appear in logs — even at DEBUG level. Redact or omit them entirely.

```python
# Never do this
logger.debug("Authenticating with token=%s", api_token)
logger.info("Processing card number %s", card_number)

# Log the presence or shape, not the value
logger.debug("Authenticating — token present: %s", bool(api_token))
logger.info("Processing payment card ending in %s", card_number[-4:])
```

### Keep messages machine-friendly in structured contexts

When emitting JSON logs, prefer `key=value` pairs as separate fields rather than interpolating everything into the message string. Log aggregators can then filter and aggregate on individual fields.

```python
# Less useful in structured logging — all context buried in the string
logger.info("Order %s for customer %s totalling $%.2f shipped", order_id, customer_id, total)

# Better — fields are individually indexable
logger.info(
    "Order shipped",
    extra={"order_id": order_id, "customer_id": customer_id, "total_usd": total},
)
```

---

## 5. Configuration Patterns

### `basicConfig` — for scripts and simple tools

Fine for one-off scripts. Call it **once**, at the entry point, before any logging calls.

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
```

### `dictConfig` — for applications (recommended)

`logging.config.dictConfig` keeps your full logging setup in one declarative dictionary, making it easy to version-control, load from YAML/TOML, and swap between environments.

```python
import logging.config

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {
            "format": "%(asctime)s %(levelname)-8s %(name)s: %(message)s",
            "datefmt": "%Y-%m-%dT%H:%M:%S",
        },
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "fmt": "%(asctime)s %(levelname)s %(name)s %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "standard",
            "stream": "ext://sys.stdout",
        },
    },
    "root": {
        "level": "INFO",
        "handlers": ["console"],
    },
    # Silence noisy third-party loggers
    "loggers": {
        "urllib3": {"level": "WARNING"},
        "boto3": {"level": "WARNING"},
    },
}

logging.config.dictConfig(LOGGING)
```

### `fileConfig` — legacy

`fileConfig` reads INI-style `.cfg` files. Prefer `dictConfig` for all new projects — it is more expressive and easier to construct programmatically.

---

## 6. Structured Logging

Plain-text logs are hard to query at scale. In production, emit **structured (JSON) logs** so that log aggregators (Datadog, Loki, CloudWatch, etc.) can index fields directly.

### Option A: `python-json-logger` (stdlib wrapper)

Drop-in replacement for the standard formatter — keeps the familiar `logging` API.

```bash
uv add python-json-logger
```

```python
import logging
import logging.config

logging.config.dictConfig({
    "version": 1,
    "formatters": {
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "fmt": "%(asctime)s %(levelname)s %(name)s %(message)s",
        }
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "json"}
    },
    "root": {"level": "INFO", "handlers": ["console"]},
})

logger = logging.getLogger(__name__)
logger.info("Order processed", extra={"order_id": 42, "amount_usd": 9.99})
# {"asctime": "2026-03-19T12:00:00", "levelname": "INFO", "name": "myapp.orders",
#  "message": "Order processed", "order_id": 42, "amount_usd": 9.99}
```

### Option B: `structlog` (standalone structured logger)

`structlog` is a more opinionated library with a pipeline-based processor chain. It's a great choice when you want context binding, lazy rendering, or full control over the log pipeline.

```bash
uv add structlog
```

```python
import structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)

log = structlog.get_logger()
log.info("user_created", user_id=42, email="alice@example.com")
# {"event": "user_created", "user_id": 42, "email": "alice@example.com",
#  "level": "info", "timestamp": "2026-03-19T12:00:00Z"}
```

Use `structlog.contextvars.bind_contextvars(request_id=req_id)` at the start of a request to automatically include context in every subsequent log call within that scope.

---

## 7. Worked Example

A complete setup for a small application using stdlib logging with `dictConfig` and JSON output in production.

```python
# myapp/logging_config.py
import logging.config
import os

def configure_logging() -> None:
    """Configure logging for the application.

    Uses JSON formatter in production (LOG_FORMAT=json) and a
    human-readable format in development.
    """
    log_format = os.getenv("LOG_FORMAT", "text")
    log_level = os.getenv("LOG_LEVEL", "INFO").upper()

    formatters: dict = {
        "text": {
            "format": "%(asctime)s %(levelname)-8s %(name)s: %(message)s",
            "datefmt": "%Y-%m-%dT%H:%M:%S",
        },
    }

    if log_format == "json":
        formatters["json"] = {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "fmt": "%(asctime)s %(levelname)s %(name)s %(message)s",
        }

    active_formatter = "json" if log_format == "json" else "text"

    logging.config.dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": formatters,
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "formatter": active_formatter,
                "stream": "ext://sys.stdout",
            },
        },
        "root": {"level": log_level, "handlers": ["console"]},
        "loggers": {
            "urllib3": {"level": "WARNING"},
            "httpx": {"level": "WARNING"},
        },
    })
```

```python
# myapp/__main__.py
from myapp.logging_config import configure_logging

configure_logging()   # call once, before anything else

# myapp/orders/processor.py
import logging

logger = logging.getLogger(__name__)

def process_order(order_id: int) -> bool:
    logger.info("Processing order %s", order_id)
    try:
        # ... business logic ...
        logger.debug("Order %s validated, submitting to payment gateway", order_id)
        logger.info("Order %s completed successfully", order_id)
        return True
    except PaymentError as exc:
        logger.error("Payment failed for order %s: %s", order_id, exc, exc_info=True)
        return False
```

---

## Further Reading

- [Python Docs — Logging HOWTO](https://docs.python.org/3/howto/logging.html)
- [Python Docs — logging.config](https://docs.python.org/3/library/logging.config.html)
- [structlog documentation](https://www.structlog.org/)
- [python-json-logger](https://github.com/madzak/python-json-logger)
