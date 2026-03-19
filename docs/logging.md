# Python Logging Best Practices

A concise guide to configuring and using Python's logging system effectively in production code.

---

## Table of Contents

1. [Use `logging`, Not `print()`](#1-use-logging-not-print)
2. [Name Loggers with `__name__`](#2-name-loggers-with-__name__)
3. [Log Levels](#3-log-levels)
4. [Configuration Patterns](#4-configuration-patterns)
5. [Structured Logging](#5-structured-logging)
6. [Worked Example](#6-worked-example)

---

## 1. Use `logging`, Not `print()`

`print()` has no concept of severity, destination, or filtering — it just writes to stdout and disappears in production. The `logging` module gives you:

- **Severity levels** — suppress noisy debug output without changing code
- **Routing** — send logs to files, syslog, external services, or multiple handlers at once
- **Structured context** — attach timestamps, module names, and custom fields
- **Runtime control** — change verbosity without redeploying

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

## 4. Configuration Patterns

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

## 5. Structured Logging

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

## 6. Worked Example

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
