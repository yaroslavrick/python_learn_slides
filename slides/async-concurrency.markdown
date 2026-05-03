---
title: Async & concurrency
---

![Three async tasks on an event loop with awaits](/assets/images/topics/async-concurrency.svg)
<!-- .element: class="title-illustration" -->

# Async & concurrency

`asyncio`, threads, processes — when each pays off.

---

## The three concurrency models

| | Use when | Stdlib module |
| --- | --- | --- |
| **`asyncio`** | I/O-bound, many connections, cooperative | `asyncio` |
| **Threads** | I/O-bound but with sync libraries | `threading`, `concurrent.futures` |
| **Processes** | CPU-bound (escape the GIL) | `multiprocessing`, `concurrent.futures` |

Picking the wrong one wastes effort. We'll cover each.

---

## The GIL, in one sentence

The **Global Interpreter Lock** ensures only **one thread executes Python bytecode at a time** in CPython. Threads can't speed up CPU-bound code; they can speed up I/O (the lock is released during blocking calls).

Free-threaded CPython (PEP 703, Python 3.13+ optional) removes the GIL — but ecosystem support is still settling.

---

## `asyncio` — the basics

```python
import asyncio

async def hello(name):
    print(f"hi, {name}")
    await asyncio.sleep(1)
    print(f"bye, {name}")

asyncio.run(hello("Alice"))
# hi, Alice
# (1 second)
# bye, Alice
```

- `async def` defines a coroutine
- `await` suspends until the awaited thing is ready
- `asyncio.run(...)` is the entry point — start the event loop, run, exit

---

## Concurrency — `gather`

Run several coroutines at the same time:

```python
async def fetch(name, delay):
    print(f"{name} start")
    await asyncio.sleep(delay)
    print(f"{name} done")
    return name

async def main():
    results = await asyncio.gather(
        fetch("A", 1),
        fetch("B", 1),
        fetch("C", 1),
    )
    return results

asyncio.run(main())
# A start, B start, C start
# (1 second total — all three sleep concurrently)
# A done, B done, C done
# → ['A', 'B', 'C']
```

Three 1-second sleeps → 1 second total, not 3.

---

## TaskGroup (3.11+)

`asyncio.gather` works, but `TaskGroup` has cleaner exception handling:

```python
async def main():
    async with asyncio.TaskGroup() as tg:
        a = tg.create_task(fetch("A", 1))
        b = tg.create_task(fetch("B", 1))
    return a.result(), b.result()
```

If any task raises, the group cancels the others and re-raises an `ExceptionGroup`. The block "owns" the lifetimes.

---

## Async I/O — `httpx` example

```python
import httpx, asyncio

async def fetch(url):
    async with httpx.AsyncClient() as client:
        r = await client.get(url)
        return r.status_code

async def main():
    urls = ["https://httpbin.org/get"] * 50
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(
            *(client.get(u) for u in urls)
        )
    return [r.status_code for r in results]

asyncio.run(main())
# [200, 200, 200, ...]   (50 requests, ~one-request-of-time)
```

50 sequential requests → 50× the time. 50 concurrent → ~1× the time.

---

## When async **doesn't** help

```python
async def fib(n):
    if n < 2:
        return n
    return await fib(n - 1) + await fib(n - 2)

await fib(35)            # ← still single-threaded, still slow
```

`asyncio` doesn't parallelize CPU work. The event loop runs one task at a time; `await` only helps when the awaited operation **gives control back** (typically I/O).

For CPU-heavy work: **multiprocessing** or move to a real worker queue.

---

## Threads — `concurrent.futures.ThreadPoolExecutor`

For I/O with sync libraries (or libraries that don't have an async equivalent):

```python
from concurrent.futures import ThreadPoolExecutor
import requests

urls = [...] * 50

with ThreadPoolExecutor(max_workers=10) as ex:
    responses = list(ex.map(requests.get, urls))
# 50 requests, ~5x faster than sequential
```

Threads release the GIL during `socket.recv()` etc., so I/O-bound work scales. CPU-bound work? Still serial.

---

## Processes — `ProcessPoolExecutor`

For CPU-bound work:

```python
from concurrent.futures import ProcessPoolExecutor

def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

with ProcessPoolExecutor() as ex:
    results = list(ex.map(fib, [33, 34, 35, 36]))
# All four run on separate cores in parallel
```

Each process has its own GIL → real parallelism. But:

- Inter-process communication is **expensive** (pickling)
- Startup cost per process
- Shared state needs `multiprocessing.Queue` / `Manager`

For "embarrassingly parallel" CPU work, this works. For everything else, it's awkward.

---

## `asyncio.to_thread` — escape to a thread

When you need to call a sync function from async code without blocking the event loop:

```python
import asyncio
from PIL import Image

async def thumbnail(path: str):
    return await asyncio.to_thread(make_thumbnail, path)

def make_thumbnail(path: str) -> bytes:
    img = Image.open(path)
    img.thumbnail((100, 100))
    ...                                      # CPU-bound, sync
```

`to_thread` runs the sync function in a thread pool. The event loop keeps spinning for other tasks.

---

## Cancellation

```python
async def main():
    task = asyncio.create_task(slow_op())
    await asyncio.sleep(0.5)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("cancelled")
```

Cancellation in asyncio is **cooperative** — the task gets a `CancelledError` raised at its next `await`. Long CPU loops in async code don't notice cancellation; they need explicit `await asyncio.sleep(0)` checkpoints or to be moved to a thread.

---

## Timeouts

```python
async def main():
    try:
        async with asyncio.timeout(5):       # 3.11+
            await slow_op()
    except TimeoutError:
        print("took too long")
```

`asyncio.timeout` is a context manager — clean, composable, propagates cancellation properly. Older code uses `asyncio.wait_for(coro, timeout=5)`.

---

## Async + sync interop

A frequent gotcha:

```python
# Inside an async view:
result = sync_db_call()              # blocks the event loop ❌
```

```python
# Better:
result = await asyncio.to_thread(sync_db_call)         # ✓ in a thread
```

Or use an **async-native** library (`asyncpg`, `httpx`, `aioredis`). Mixing sync and async carelessly will tank the event loop's throughput.

---

## Real-world picks

| Problem | Tool |
| --- | --- |
| Web app handling 1000 concurrent requests | FastAPI + asyncio + async DB driver |
| Crawling 500 URLs nightly | `asyncio.gather` + `httpx.AsyncClient` |
| Image batch resizing | `ProcessPoolExecutor` |
| One-off "scrape this list" script | `ThreadPoolExecutor` + `requests` |
| Long-running message consumer | `asyncio` event loop, or a sync worker per process |
| ML training | `multiprocessing` (or, really, a GPU + framework) |

Concurrency choice flows from the **shape** of the work, not the latest hype.

---

## Common mistakes

- **Calling a sync blocking function from async code** — kills throughput. Use `to_thread` or an async library.
- **Using `time.sleep` in async** — same problem. Use `await asyncio.sleep`.
- **Creating tasks without keeping a reference** — they can be GC'd before they finish. Hold the `Task` object or use a `TaskGroup`.
- **Catching `BaseException`** — swallows `CancelledError`. Catch `Exception` instead.
- **Mixing `requests` and `httpx.AsyncClient`** without `to_thread` — same blocking problem.

---

## When to skip concurrency entirely

- Your app is fast enough already
- One database does most of the work — let the DB handle parallelism
- You'd be adding complexity for a 5% win

Profile first. Most "slow" Python is single-query slow, not concurrency-starved.

---

## What's next

- **Git** — version control basics
- **Docker** — packaging for deploy
- **Deployment** — putting it all together
