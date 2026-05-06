---
title: Django
---

![Django stack](/assets/images/topics/django.svg)
<!-- .element: class="title-illustration" -->

# Django

The batteries-included web framework.

---

## Why Django?

- **Batteries included** — ORM, admin, auth, forms, templates, tests, migrations, all in the box
- **Convention over configuration** — sensible defaults; predictable structure
- **The admin** — a CRUD UI for your models, free, in five minutes
- **Mature** — 20+ years, security-fixed, well-documented

If your app needs a database, users, and a UI, Django gets you there fastest.

---

## Why not Django?

- **Full-stack opinion** — if you only need a JSON API, FastAPI is leaner
- **Sync-first** — async support exists but isn't the primary path (yet)
- **Magic** — meta-classes for models, signals, autoloading; can be opaque on first contact

For modern API-only / async-heavy services, **FastAPI** is often the better fit. We cover it next.

---

## Start a project

```bash
uv init my-site
cd my-site
uv add django
uv run django-admin startproject mysite .
```

Result:

```
my-site/
├── pyproject.toml
├── uv.lock
├── manage.py
└── mysite/
    ├── __init__.py
    ├── settings.py        # configuration
    ├── urls.py            # URL routing
    ├── asgi.py / wsgi.py  # entry points for servers
    └── views.py           # (you'll add this)
```

`manage.py` is the project's command center.

---

## Run the dev server

```bash
uv run python manage.py runserver
# Watching for file changes with StatReloader
# Performing system checks...
# Django version 5.x, using settings 'mysite.settings'
# Starting development server at http://127.0.0.1:8000/
```

Open <http://127.0.0.1:8000/> — Django's "rocket" welcome page.

The server auto-reloads on file changes.

---

## manage.py — the command center

Common commands:

```bash
uv run python manage.py runserver           # dev server
uv run python manage.py startapp blog       # create an app
uv run python manage.py makemigrations      # generate migrations
uv run python manage.py migrate             # apply them
uv run python manage.py createsuperuser     # admin account
uv run python manage.py shell               # Django-aware REPL
uv run python manage.py test                # run tests (or pytest)
```

`manage.py <command> --help` lists every option.

---

## Apps vs the project

A **project** is your site (the directory with `settings.py`).
An **app** is a self-contained feature (`blog`, `accounts`, `payments`).

```bash
uv run python manage.py startapp blog
```

```
blog/
├── __init__.py
├── admin.py        # admin registration
├── apps.py         # app config
├── migrations/     # auto-generated DB migrations
├── models.py       # data models (ORM classes)
├── tests.py        # tests for this app
└── views.py        # request handlers
```

Register it in `settings.py` → `INSTALLED_APPS`.

---

## settings.py — INSTALLED_APPS

The first thing you'll edit is the apps list:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",                          # ← add your apps here
]
```

Each app contributes its models, migrations, templates, and admin registrations.

--

## settings.py — database and locale

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}

LANGUAGE_CODE = "en-us"
TIME_ZONE = "UTC"
USE_TZ = True                        # store timestamps in UTC, render in TZ
```

SQLite is fine for development. Production gets Postgres (next slide).

--

## settings.py — for production

```python
import os

DEBUG = os.environ.get("DJANGO_DEBUG") == "1"
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]    # never in source!
ALLOWED_HOSTS = os.environ.get("DJANGO_HOSTS", "").split(",")

# Use Postgres in production
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ["DB_NAME"],
        "USER": os.environ["DB_USER"],
        "PASSWORD": os.environ["DB_PASSWORD"],
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": "5432",
    }
}
```

Reach for `django-environ` or `pydantic-settings` for cleaner env-var handling.

---

## The request / response cycle

```
HTTP request
   ↓
ASGI/WSGI server (gunicorn / uvicorn)
   ↓
URL dispatcher (urls.py)         ← we'll cover this in URLs
   ↓
Middleware (auth, sessions, CSRF, ...)
   ↓
View (a function or class)       ← Views
   ↓
ORM / business logic             ← Models, refactoring
   ↓
Template rendering               ← Templates
   ↓
HTTP response
```

Each step is a deck of its own.

---

## A complete tiny example

```python
# mysite/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("hello/", views.hello),
]
```

```python
# mysite/views.py
from django.http import HttpResponse

def hello(request):
    return HttpResponse("Hello, world!")
```

`uv run python manage.py runserver` → visit `/hello/` → "Hello, world!".

That's the whole loop. Everything else builds on this.

---

## The admin — five-minute CRUD UI

```python
# blog/models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    published = models.BooleanField(default=False)
```

```python
# blog/admin.py
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

`makemigrations`, `migrate`, `createsuperuser` → log in at `/admin/` → full CRUD on Posts. No frontend code.

---

## Where Django shines

- Content sites, blogs, internal tools, dashboards
- Apps with non-trivial domain models
- Anything that benefits from the admin
- Teams that want one stack to rule them all

---

## What's next

- **Models** — ORM, migrations, relationships, querysets
- **URLs** — routing, named routes, includes
- **Views** — function- and class-based
- **Templates** — DTL, forms, ModelForms
