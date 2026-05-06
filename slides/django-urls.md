---
title: Django URLs
---

![URL pattern matching a path to a view](/assets/images/topics/django-urls.svg)
<!-- .element: class="title-illustration" -->

# Django URLs

Routing requests to views.

---

## URLconf basics

`urls.py` is a list of `URLPattern` objects. Each maps a URL path to a view.

```python
# mysite/urls.py
from django.urls import path
from blog import views

urlpatterns = [
    path("",            views.index,        name="index"),
    path("about/",      views.about,        name="about"),
    path("post/<int:id>/", views.post_detail, name="post"),
]
```

`name=` is critical — we'll use it for **reverse URL lookup** (next).

---

## Path converters

Capture values from the URL into view kwargs:

| Converter | Matches | Example |
| --- | --- | --- |
| `str` (default) | non-slash chars | `<str:slug>` → `slug="hello"` |
| `int` | integers | `<int:id>` → `id=42` |
| `slug` | `[a-z0-9_-]+` | `<slug:slug>` |
| `uuid` | UUID format | `<uuid:id>` |
| `path` | anything incl. `/` | `<path:rest>` |

```python
path("post/<int:id>/", views.post_detail)

# In the view:
def post_detail(request, id): ...    # id is an int, not a str
```

--

## Custom path converters

For domain-specific routes:

```python
# converters.py
class FourDigitYearConverter:
    regex = r"\d{4}"
    def to_python(self, value): return int(value)
    def to_url(self, value): return f"{value:04d}"

# urls.py
from django.urls import register_converter
from . import converters

register_converter(converters.FourDigitYearConverter, "yyyy")

urlpatterns = [
    path("archive/<yyyy:year>/", views.archive),
]
```

---

## `re_path` for regex

When `path` converters aren't enough:

```python
from django.urls import re_path

urlpatterns = [
    re_path(r"^posts/(?P<year>\d{4})/(?P<month>\d{2})/$", views.archive),
]
```

`<int:id>` syntax works in `path()`; named groups work in `re_path()`. Use `path` whenever possible — clearer.

---

## Splitting URLs — per-app file

A growing site keeps each app's URLs in its own file:

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = "blog"

urlpatterns = [
    path("",               views.index,       name="index"),
    path("<int:id>/",      views.detail,      name="detail"),
]
```

---

## Splitting URLs — including from the project

```python
# mysite/urls.py
from django.urls import path, include

urlpatterns = [
    path("blog/", include("blog.urls")),
    path("admin/", admin.site.urls),
]
```

`/blog/` → `blog.views.index`. `/blog/42/` → `blog.views.detail(id=42)`.

---

## Namespacing

`app_name = "blog"` in the app's `urls.py` lets you reference URLs as `"blog:detail"` — avoiding collisions with `accounts:detail` or anywhere else.

```python
# In templates:
{% raw %}{% url 'blog:detail' post.id %}{% endraw %}     {# /blog/42/ #}

# In Python:
from django.urls import reverse
reverse("blog:detail", args=[42])              # '/blog/42/'
```

Always namespace included URLconfs; don't rely on top-level names.

---

## Reverse URL lookup

**Hard-coding URLs in templates and views is fragile** — when routes change, you fix them everywhere.

```python
# Bad
return redirect("/blog/42/")

# Good
return redirect("blog:detail", id=42)

# Or build a URL string:
from django.urls import reverse
url = reverse("blog:detail", args=[42])        # '/blog/42/'
url = reverse("blog:detail", kwargs={"id": 42})
```

Routes can move; their **names** stay stable.

---

## In templates

```django
{% raw %}<a href="{% url 'blog:detail' post.id %}">{{ post.title }}</a>
<a href="{% url 'blog:index' %}">All posts</a>
<form action="{% url 'blog:create' %}" method="post">
  {% csrf_token %}
  ...
</form>{% endraw %}
```

`{% raw %}{% url ... %}{% endraw %}` is the template equivalent of `reverse()`. Same name, same args.

---

## Class-based view URLs

Class-based views need `.as_view()`:

```python
# blog/urls.py
from django.urls import path
from .views import PostListView, PostDetailView

urlpatterns = [
    path("",               PostListView.as_view(),   name="list"),
    path("<int:pk>/",      PostDetailView.as_view(), name="detail"),
]
```

`.as_view()` returns the request-handler callable. Don't forget the `()`.

---

## Including third-party app URLs

Most third-party apps ship a urls module:

```python
# mysite/urls.py
urlpatterns = [
    path("admin/",       admin.site.urls),
    path("accounts/",    include("django.contrib.auth.urls")),
    path("api/",         include("blog.api.urls")),
    path("docs/",        include("rest_framework.urls")),
]
```

Each `include()` mounts a sub-tree at the prefix.

---

## Static and media files

In dev:

```python
# mysite/urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path("admin/", admin.site.urls),
    # ... your routes
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

In production, your CDN / web server (nginx, Caddy) serves these directly — Django doesn't.

---

## Testing URL resolution

```python
# tests/test_urls.py
from django.urls import reverse, resolve

def test_post_detail_url():
    url = reverse("blog:detail", args=[42])
    assert url == "/blog/42/"

    match = resolve("/blog/42/")
    assert match.view_name == "blog:detail"
    assert match.kwargs == {"id": 42}
```

`resolve()` is the inverse of `reverse()` — useful for testing routing config without running the view.

---

## What's next

- **Views** — what runs when a route matches
- **Templates** — rendering the response
- **DRF** — separate URL patterns for APIs
