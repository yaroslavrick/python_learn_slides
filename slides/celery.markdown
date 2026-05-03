---
title: Celery
---

![Web app, broker, and worker pool](/assets/images/topics/celery.svg)
<!-- .element: class="title-illustration" -->

# Celery

Background jobs for Django (and any Python).

---

## Why a task queue?

Some work doesn't belong in the request/response cycle:

- **Email / SMS** — slow, can fail
- **Image / video processing** — long-running
- **Webhooks to third parties** — flaky networks
- **Scheduled jobs** — cleanup, reports, data sync
- **Fan-out** — one event, many consumers

Doing these inline ties your web worker up. Push them to a queue, return a response immediately, process them in a worker pool.

---

## Anatomy

```
┌─────────────┐    publish   ┌──────────────┐   pull   ┌────────────┐
│ Django view │ ───────────→ │   Broker     │ ──────→  │  Workers   │
│ (or script) │              │ (Redis,      │          │ (your code)│
└─────────────┘              │  RabbitMQ)   │          └────────────┘
                             └──────────────┘                │
                             ┌──────────────┐  results       │
                             │   Backend    │ ←──────────────┘
                             │ (Redis, DB)  │
                             └──────────────┘
```

- **Broker** — message bus (Redis or RabbitMQ)
- **Backend** — stores task results (often Redis)
- **Workers** — processes that pull tasks and run them

---

## Install

```bash
uv add celery redis
# Or for Redis broker:
uv add "celery[redis]"
```

You'll also need Redis running. Locally with Docker:

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

---

## Configure

```python
# mysite/celery.py
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")

app = Celery("mysite")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()
```

```python
# mysite/__init__.py
from .celery import app as celery_app
__all__ = ("celery_app",)
```

```python
# settings.py
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/1"
CELERY_TASK_SERIALIZER = "json"
CELERY_TIMEZONE = "UTC"
```

`autodiscover_tasks()` finds `tasks.py` modules in every installed app.

---

## Defining a task

```python
# blog/tasks.py
from celery import shared_task
from django.core.mail import send_mail

@shared_task
def send_post_notification(post_id):
    from .models import Post                 # avoid app-loading issues
    post = Post.objects.get(pk=post_id)
    for sub in post.subscribers.all():
        send_mail(
            f"New post: {post.title}",
            post.body,
            "no-reply@example.com",
            [sub.email],
        )
```

`@shared_task` (preferred over `@app.task`) decouples the task definition from a specific Celery app.

---

## Calling tasks

```python
# blog/views.py
from .tasks import send_post_notification

def publish(request, post_id):
    post = Post.objects.get(pk=post_id)
    post.published = True
    post.save()
    send_post_notification.delay(post.id)        # → enqueued
    return redirect("blog:detail", id=post.id)
```

`task.delay(*args)` — enqueue with default options.
`task.apply_async(args=[...], countdown=60)` — full control.

**Pass IDs, not model instances.** Instances aren't serializable; the worker fetches the row when it runs.

---

## Running workers

```bash
uv run celery -A mysite worker --loglevel=info
# [tasks]
#   . blog.tasks.send_post_notification
# [INFO/MainProcess] Connected to redis://localhost:6379/0
# [INFO/MainProcess] mingle: searching for neighbors
# [INFO/MainProcess] celery@mac.local ready.
```

In production, run multiple workers (`-c 4` concurrency, or one process per CPU).

---

## Retries

```python
@shared_task(bind=True, autoretry_for=(httpx.HTTPError,),
             retry_backoff=True, retry_kwargs={"max_retries": 5})
def call_webhook(self, url, payload):
    httpx.post(url, json=payload, timeout=10)
```

- `autoretry_for=(...)` — auto-retry on these exception types
- `retry_backoff=True` — exponential delay
- `max_retries=5` — give up after 5 attempts

Manual retry inside the task:

```python
@shared_task(bind=True, max_retries=3)
def my_task(self, x):
    try:
        risky_op(x)
    except ConnectionError as e:
        raise self.retry(exc=e, countdown=2 ** self.request.retries)
```

---

## Periodic tasks — Celery Beat

```python
# settings.py
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "cleanup-drafts-daily": {
        "task": "blog.tasks.cleanup_drafts",
        "schedule": crontab(hour=3, minute=0),
    },
    "warm-cache-every-minute": {
        "task": "ops.tasks.warm_cache",
        "schedule": 60.0,             # seconds
    },
}
```

Run beat alongside the worker:

```bash
uv run celery -A mysite beat --loglevel=info
```

For dynamic schedules edited in the admin, use `django-celery-beat`.

---

## Task results

```python
result = send_post_notification.delay(post.id)
result.id                # task UUID
result.ready()           # bool
result.get(timeout=5)    # block until done; raises if it failed
result.successful()      # only after completion
result.state             # 'PENDING' | 'STARTED' | 'SUCCESS' | 'FAILURE'
```

Don't `.get()` from a request — defeats the point. For "did it finish?" polling, use the task ID and `AsyncResult`.

---

## Chaining and grouping

```python
from celery import chain, group, chord

# Run sequentially, pipe results
result = chain(
    fetch.s("https://api.example.com/data"),
    transform.s(),
    save.s(),
).apply_async()

# Run in parallel
header = group(fetch.s(url) for url in urls)
result = header.apply_async()
result.get()                                  # list of results

# Run group, then a callback with all results
result = chord(
    (fetch.s(u) for u in urls),
    summarize.s(),
).apply_async()
```

`.s(args)` creates a "signature" — a deferred call.

---

## Testing tasks

In tests, run tasks **synchronously**:

```python
# settings_test.py
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True
```

Tasks run inline; `.delay()` returns an already-ready result.

```python
def test_publish_notifies(db):
    post = PostFactory(published=False)
    post.publish()                              # internally calls .delay()
    assert SentEmail.objects.count() == post.subscribers.count()
```

For more realistic tests, use `pytest-celery` to run a real worker in-process.

---

## When to use Celery — and when not

| Use Celery for... | Don't use Celery for... |
| --- | --- |
| Heavy / slow tasks | Fire-and-forget where loss is OK (use a goroutine-style background thread) |
| Tasks needing retries / backoff | Simple cron jobs (use systemd timers / cron + manage.py) |
| Fan-out from one event | A queue you only consume from once a day |
| Workflows (chain / chord) | Real-time message passing (use channels, websockets) |

Celery is operationally heavy — broker + workers + monitoring. Justify it.

---

## Lighter alternatives

- **`rq`** — Redis-backed queue, much simpler API, no broker complexity.
- **`huey`** — like rq, with built-in periodic scheduling.
- **`dramatiq`** — modern alternative to Celery, simpler config.
- **Django `Q2`** — Django-integrated queue, embedded.

If your queue is "send 100 emails / hour and clean up nightly", `rq` may be all you need.

---

## What's next

- **DRF** — exposing your services as APIs
- **GraphQL** — alternative API style
