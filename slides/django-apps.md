---
title: Django apps
---

![A project with multiple apps inside](/assets/images/topics/django-apps.svg)
<!-- .element: class="title-illustration" -->

# Django apps

Apps as reusable units. Packaging an app for distribution.

---

## Project vs app

| | What it is |
| --- | --- |
| **Project** | The site as a whole (settings, top-level URLs, deploy config) |
| **App** | A self-contained feature with its own models, views, templates |

A project usually has many apps. An app **may** be reusable across projects (e.g. `django-allauth`, `django-rest-framework`).

---

## What an app contains

```
blog/
├── __init__.py
├── apps.py             # AppConfig — name, ready hook, label
├── admin.py            # admin registrations
├── models.py
├── views.py
├── urls.py             # app's URL patterns
├── forms.py
├── managers.py         # custom QuerySets / Managers
├── signals.py          # signal handlers
├── templates/blog/     # HTML templates (namespaced)
├── static/blog/        # CSS, JS, images (namespaced)
├── templatetags/       # custom template tags + filters
├── migrations/
├── management/
│   └── commands/       # custom manage.py commands
└── tests/
```

Not all directories are required — start small, grow as needed.

---

## `apps.py` — the AppConfig

```python
# blog/apps.py
from django.apps import AppConfig

class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"
    verbose_name = "Blog posts"

    def ready(self):
        # Import signal handlers so they're registered
        from . import signals
```

`ready()` runs once per process at startup — the right place to register signals or one-time setup.

---

## Splitting a feature into an app

Rule of thumb: an app is a unit you could **move to its own package** with minor changes. If two features touch the same models, they're probably one app. If a feature has its own models with little overlap, it's a candidate for its own app.

Examples of well-scoped apps:

- `accounts` — User model, signup, profile
- `billing` — subscriptions, invoices, payments
- `blog` — Post, Comment, Tag
- `notifications` — Notification model + dispatch
- `core` — domain-agnostic utilities

If you're not sure, start with one app and split later. It's cheap.

---

## Inter-app dependencies

Apps can import each other's models — but be deliberate. A clean layering:

```
core ←─────── (everyone)
accounts ←─── billing, blog, notifications
billing ←──── notifications
blog ←─────── notifications
```

Avoid cycles. If `blog` imports from `notifications` and `notifications` imports from `blog`, refactor — extract a shared interface to `core` or use **signals** for the dependency.

---

## Signals — decoupled events

```python
# blog/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Post

@receiver(post_save, sender=Post)
def notify_subscribers(sender, instance, created, **kwargs):
    if created and instance.published:
        from notifications.tasks import send_post_alert
        send_post_alert.delay(instance.id)
```

Wire up in `apps.py`'s `ready()`. Signals are great for cross-app side effects without imports.

**Caveat**: signals can become hard to trace. Prefer explicit calls when the event has one obvious caller; reserve signals for true many-to-many event flows.

---

## Custom management commands

```python
# blog/management/commands/cleanup_drafts.py
from django.core.management.base import BaseCommand
from blog.models import Post

class Command(BaseCommand):
    help = "Delete drafts older than N days"

    def add_arguments(self, parser):
        parser.add_argument("--days", type=int, default=30)

    def handle(self, *args, **options):
        cutoff = timezone.now() - timedelta(days=options["days"])
        count, _ = Post.objects.filter(published=False, created_at__lt=cutoff).delete()
        self.stdout.write(self.style.SUCCESS(f"Deleted {count} drafts"))
```

Run with:

```bash
uv run python manage.py cleanup_drafts --days=60
# Deleted 17 drafts
```

Schedule with cron, systemd timers, or Celery beat.

---

## Reusing apps

Two main paths:

1. **Drop in via PyPI** — install someone else's app (`django-allauth`, `django-import-export`).
2. **Package your own** — turn an app into an installable distribution.

The structure is the same — the app is just shipped as a Python package.

---

## Packaging an app — minimal structure

```
django-blog-app/
├── pyproject.toml
├── README.md
├── LICENSE
├── src/
│   └── django_blog_app/
│       ├── __init__.py
│       ├── apps.py
│       ├── models.py
│       ├── views.py
│       └── ...
└── tests/
```

`src/` layout keeps your tests from accidentally importing the in-tree copy.

--

## `pyproject.toml`

```toml
[project]
name = "django-blog-app"
version = "0.1.0"
description = "Reusable blog app"
dependencies = ["django>=5.0"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

`hatchling` is the modern default build backend. Replace it with `setuptools` if you have a reason; for new projects, hatchling is fine.

--

## Packaging — entry point

`apps.py`:

```python
class DjangoBlogAppConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "django_blog_app"            # the importable Python package name
    label = "blog"                      # short label, used in DB tables, perms
    verbose_name = "Blog"
```

Users install with `uv add django-blog-app`, add `"django_blog_app"` to `INSTALLED_APPS`, and include URLs:

```python
# project urls.py
path("blog/", include("django_blog_app.urls")),
```

Then `migrate` to create tables.

---

## Reusable app — settings hooks

Don't hard-code; let the host project override.

```python
# django_blog_app/conf.py
from django.conf import settings as project_settings

POSTS_PER_PAGE = getattr(project_settings, "BLOG_POSTS_PER_PAGE", 10)
ALLOW_COMMENTS = getattr(project_settings, "BLOG_ALLOW_COMMENTS", True)
```

Document these in your `README` so users know what they can override.

---

## Migrations in reusable apps

Ship the migrations directory. The host project gets them via `python manage.py migrate`.

```
src/django_blog_app/
└── migrations/
    ├── __init__.py
    └── 0001_initial.py
```

For changes to user-customizable models, prefer `swappable = "BLOG_POST_MODEL"` so projects can override the model — like Django's `AUTH_USER_MODEL`.

---

## Testing a reusable app

```python
# tests/conftest.py
import django
from django.conf import settings

def pytest_configure():
    settings.configure(
        DATABASES={"default": {
            "ENGINE": "django.db.backends.sqlite3",
            "NAME": ":memory:",
        }},
        INSTALLED_APPS=[
            "django.contrib.auth",
            "django.contrib.contenttypes",
            "django_blog_app",
        ],
        ROOT_URLCONF="tests.urls",
    )
    django.setup()
```

Then run with `uv run pytest`. The app should be testable **without a project around it**.

---

## A clean app checklist

- [ ] Self-contained `models.py` — no imports from sibling apps in models
- [ ] Templates and static files namespaced (`templates/blog/`, `static/blog/`)
- [ ] `urls.py` with `app_name = "..."` set
- [ ] Custom permissions defined in `Meta`
- [ ] `apps.py` with `ready()` for signal registration
- [ ] Tests live under `tests/` and run with `pytest`
- [ ] No `print()` calls — use `logging`
- [ ] Settings keys documented in README

The more boxes you tick, the easier the app is to reuse.

---

## What's next

- **Refactoring Django** — keep models fat, views thin, services for the rest
- **BDD** — testing through user-story scenarios
- **DRF** — exposing your apps as APIs
