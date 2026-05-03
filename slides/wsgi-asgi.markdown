---
title: WSGI and ASGI
---

![Server speaking WSGI or ASGI to a Python framework](/assets/images/topics/wsgi-asgi.svg)
<!-- .element: class="title-illustration" -->

# WSGI & ASGI

The protocols Python web servers and frameworks speak.

---

## What WSGI is

**WSGI** (PEP 3333) — Web Server Gateway Interface. The classic, sync-only Python web protocol since 2003.

A WSGI app is a callable:

```python
def app(environ, start_response):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Hello, world!"]
```

- `environ` — request data (a dict)
- `start_response` — callback for status + headers
- Return value — iterable of bytes

That's it. The whole spec.

---

## Why a protocol matters

WSGI lets **any** Python web framework work with **any** Python web server:

```
[gunicorn] ←─── WSGI ───→ [Django / Flask / Pyramid / your own]
[uWSGI]    ←─── WSGI ───→ [...]
[mod_wsgi] ←─── WSGI ───→ [...]
```

You can swap the server without touching app code, and vice versa. WSGI is the contract.

---

## What ASGI is

**ASGI** (Asynchronous Server Gateway Interface) — the async successor.

```python
async def app(scope, receive, send):
    if scope["type"] != "http":
        return
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"text/plain")],
    })
    await send({
        "type": "http.response.body",
        "body": b"Hello, world!",
    })
```

- `scope` — connection info (similar to WSGI's `environ`)
- `receive` / `send` — async callbacks

ASGI handles **HTTP, WebSockets, lifespan events** — WSGI was HTTP-only.

---

## WSGI vs ASGI

| | WSGI | ASGI |
| --- | --- | --- |
| Sync / async | Sync only | Both |
| HTTP | Yes | Yes |
| WebSockets | No | Yes |
| Long-lived connections | Hard | Native |
| Servers | gunicorn, uWSGI | uvicorn, hypercorn, daphne |
| Frameworks | Django, Flask, Pyramid | FastAPI, Starlette, Django (3+) |

A modern Django app can run on **either** — pick by your needs.

---

## gunicorn — the WSGI workhorse

```bash
uv add gunicorn
uv run gunicorn mysite.wsgi:application --workers 4
# [INFO] Listening at: http://0.0.0.0:8000
# [INFO] Using worker: sync
# [INFO] Booting worker with pid: 12345
```

Common flags:

```bash
gunicorn mysite.wsgi:application \
  --workers 4 \
  --bind 0.0.0.0:8000 \
  --timeout 30 \
  --access-logfile -
```

Workers are forked processes. Tune to ~`2 * CPU + 1` as a starting point.

---

## uvicorn — the ASGI workhorse

```bash
uv add uvicorn
uv run uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
# INFO:     Started server process [12345]
# INFO:     Waiting for application startup.
# INFO:     Application startup complete.
# INFO:     Uvicorn running on http://0.0.0.0:8000
```

For **production**, run uvicorn behind gunicorn (process supervisor):

```bash
gunicorn main:app -k uvicorn.workers.UvicornWorker --workers 4
```

You get gunicorn's robust process management with uvicorn's async event loop.

---

## Django — both worlds

`django-admin startproject` creates **both** entry points:

```python
# mysite/wsgi.py
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()

# mysite/asgi.py
from django.core.asgi import get_asgi_application
application = get_asgi_application()
```

Run either:

```bash
gunicorn mysite.wsgi:application                                 # WSGI
uvicorn mysite.asgi:application                                  # ASGI
```

If your app uses async views or WebSockets (`channels`), pick ASGI. Otherwise WSGI is simpler.

---

## FastAPI — ASGI only

FastAPI is built on Starlette, an ASGI framework. There's no WSGI entry point.

```bash
uv run fastapi dev main.py                  # uvicorn under the hood
uv run uvicorn main:app                     # equivalent, more explicit
```

For production, run uvicorn workers under gunicorn (same recipe as Django ASGI):

```bash
gunicorn main:app -k uvicorn.workers.UvicornWorker --workers 4
```

---

## When sync vs async matters

| Workload | WSGI / sync | ASGI / async |
| --- | --- | --- |
| CRUD against DB (sync driver) | ✓ Same speed; simpler | ⚠ Don't use a sync DB driver from async |
| Many concurrent slow HTTP calls | Process per call → scales poorly | One process handles thousands |
| WebSockets / SSE / long polling | Hard or impossible | Native |
| CPU-bound work | Identical (use a worker queue) | Identical (use a worker queue) |

Async is **about I/O concurrency**. It does **not** make your code faster on CPU.

---

## A common production stack

```
        nginx / Caddy
              │
              ↓
     gunicorn (process supervisor)
              │
       ┌──────┴──────┬────────┐
       ↓             ↓        ↓
   uvicorn       uvicorn   uvicorn      ← N async workers
   worker        worker    worker
       │             │        │
       ↓             ↓        ↓
   FastAPI app  (or Django ASGI)
```

- nginx/Caddy: TLS, static files, rate limiting
- gunicorn: process supervisor, workers, restart on crash
- uvicorn: ASGI event loop per process
- Your app: handles requests

---

## Health checks and graceful shutdown

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

For load balancers:

- `/health` — liveness (is the process alive?)
- `/ready` — readiness (can we accept traffic? checks DB, cache, etc.)

Gunicorn handles `SIGTERM` for graceful shutdown — it stops accepting new requests, lets in-flight ones finish, then exits.

---

## What's next

- **Async & concurrency** — `asyncio` in depth, when threads / processes are the right tool
- **Deployment** — Docker, GitHub Actions, container orchestration
