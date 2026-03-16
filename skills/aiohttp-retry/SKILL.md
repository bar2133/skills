---
name: aiohttp-retry
description: >
  Implement HTTP request retry logic using aiohttp's native client middleware system.
  Use when the user wants to add retries to aiohttp requests, handle transient HTTP errors
  (429, 5xx), or make async HTTP calls more resilient — without third-party retry libraries.
---

# aiohttp Retry Middleware

Add retry logic to `aiohttp` HTTP requests using the built-in client middleware system
(`ClientSession(middlewares=(...))`) — no third-party libraries needed. Requires aiohttp 3.x+.

**Announce at start:** "I'm using the aiohttp-retry skill to implement retry middleware."

---

## The Pattern

A factory function that returns a middleware. The middleware retries requests when the server
responds with a transient status code.

```python
import asyncio
import logging

from aiohttp import (
    ClientHandlerType,
    ClientMiddlewareType,
    ClientRequest,
    ClientResponse,
    ClientSession,
)

logger = logging.getLogger(__name__)


def retry_middleware(
    max_retries: int = 3,
    delay: float = 1.0,
    retryable_statuses: set[int] | None = None,
) -> ClientMiddlewareType:
    """Create a retry middleware for aiohttp ClientSession.

    Args:
        max_retries: Maximum number of attempts before giving up.
        delay: Seconds to wait between retry attempts.
        retryable_statuses: HTTP status codes that trigger a retry.
            Defaults to {429, 500, 502, 503, 504}.

    Returns:
        A middleware function compatible with ClientSession.
    """
    if retryable_statuses is None:
        retryable_statuses = {429, 500, 502, 503, 504}

    async def middleware(
        req: ClientRequest, handler: ClientHandlerType
    ) -> ClientResponse:
        last_resp: ClientResponse | None = None
        for attempt in range(1, max_retries + 1):
            resp = await handler(req)
            if resp.status not in retryable_statuses:
                return resp
            last_resp = resp
            if attempt < max_retries:
                logger.warning(
                    "Attempt %d/%d returned %d, retrying in %.1fs...",
                    attempt, max_retries, resp.status, delay,
                )
                await asyncio.sleep(delay)
        return last_resp

    return middleware
```

---

## Usage

```python
async with ClientSession(middlewares=(retry_middleware(max_retries=3, delay=1.0),)) as session:
    async with session.get("http://example.com/api/data") as resp:
        data = await resp.json()
```

Retries are transparent to the caller — every `session.get()`, `session.post()`, etc. goes
through the middleware automatically.

---

## Retryable Status Codes

| Code | Meaning | Why retry |
|------|---------|-----------|
| 429 | Too Many Requests | Rate limit; server asks you to slow down |
| 500 | Internal Server Error | Transient server bug |
| 502 | Bad Gateway | Upstream server unreachable |
| 503 | Service Unavailable | Server temporarily overloaded |
| 504 | Gateway Timeout | Upstream server timed out |

Do **not** retry 4xx client errors (except 429) — those indicate a problem with the request itself.

---

## Middleware Ordering

Middleware executes in the order listed. Order affects behavior:

```python
# Logging runs per retry attempt (retry wraps logging):
ClientSession(middlewares=(retry_middleware(), logging_middleware))

# Logging runs once regardless of retries (logging wraps retry):
ClientSession(middlewares=(logging_middleware, retry_middleware()))
```

Place retry **before** middlewares that should run on each attempt (logging, metrics),
and **after** middlewares that should run once (auth header injection).

---

## Important: Request-Level Override

Passing `middlewares` to an individual request **replaces** session-level middlewares entirely
(it does not extend them):

```python
session = ClientSession(middlewares=(retry_middleware(),))

# Uses retry middleware
await session.get("http://example.com")

# Does NOT use retry middleware — only custom_mw runs
await session.get("http://example.com", middlewares=(custom_mw,))

# To keep both, pass both explicitly
await session.get("http://example.com", middlewares=(retry_middleware(), custom_mw))
```

---

## Reference

- [aiohttp Client Middleware docs](https://docs.aiohttp.org/en/stable/client_advanced.html#aiohttp-client-middleware)
- [aiohttp Middleware Cookbook](https://docs.aiohttp.org/en/stable/client_middleware_cookbook.html)
