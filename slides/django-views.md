---
title: Django views
---

![Request to view to response](/assets/images/topics/django-views.svg)
<!-- .element: class="title-illustration" -->

# Django views

Function-based, class-based, generic.

---

## What a view is

A view is a callable that takes an `HttpRequest` and returns an `HttpResponse`.

```python
from django.http import HttpResponse

def hello(request):
    return HttpResponse("Hello, world!")
```

Wired up in `urls.py`:

```python
path("hello/", views.hello)
```

That's the entire contract. Everything else is convenience.

---

## Function-based views (FBVs)

```python
from django.shortcuts import render, get_object_or_404
from .models import Post

def post_list(request):
    posts = Post.objects.filter(published=True)
    return render(request, "blog/list.html", {"posts": posts})

def post_detail(request, id):
    post = get_object_or_404(Post, pk=id)
    return render(request, "blog/detail.html", {"post": post})
```

`render(request, template, context)` is the everyday helper.

---

## HttpRequest — what's on it

```python
def my_view(request):
    request.method                # 'GET' / 'POST' / ...
    request.GET                   # query string params (QueryDict)
    request.POST                  # form data (QueryDict)
    request.body                  # raw bytes (for JSON APIs)
    request.user                  # the authenticated user (or AnonymousUser)
    request.session               # session dict
    request.headers["X-Foo"]      # case-insensitive header access
    request.META["HTTP_USER_AGENT"]
    request.path                  # '/blog/42/'
    request.GET.get("q", "")      # safe access with default
```

---

## HttpResponse variants

```python
from django.http import HttpResponse, JsonResponse, HttpResponseRedirect
from django.shortcuts import redirect, render

return HttpResponse("plain text")
return HttpResponse("<h1>html</h1>", content_type="text/html")
return JsonResponse({"status": "ok"})
return JsonResponse([1, 2, 3], safe=False)

return redirect("blog:detail", id=42)        # by URL name
return redirect(post)                        # uses post.get_absolute_url()
return redirect("/blog/42/")                 # by path

return render(request, "tpl.html", ctx)      # template + context

# Errors
from django.http import Http404
raise Http404("Post not found")
```

---

## Handling forms in FBVs

```python
def create_post(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save()
            return redirect("blog:detail", id=post.id)
    else:
        form = PostForm()
    return render(request, "blog/form.html", {"form": form})
```

The "GET shows form, POST processes it" pattern. The classic Django flow.

---

## Class-based views (CBVs)

CBVs are classes whose methods (`get`, `post`, ...) handle HTTP verbs.

```python
from django.views import View
from django.shortcuts import render

class PostListView(View):
    def get(self, request):
        posts = Post.objects.filter(published=True)
        return render(request, "blog/list.html", {"posts": posts})

# urls.py
path("", PostListView.as_view(), name="list")
```

`.as_view()` adapts the class to Django's view protocol.

---

## Why CBVs?

- **Inheritance** — share behavior across views via mixins
- **Generic views** — common patterns (list, detail, create, update, delete) provided by Django
- **Verb dispatch** — separate methods for `get`, `post`, `put`, `delete`

```python
class PostView(View):
    def get(self, request, id):
        return render(...)
    def post(self, request, id):
        ...
        return redirect(...)
```

For very simple views, FBVs are still shorter. Use whichever reads better.

---

## Generic views — `ListView` and `DetailView`

```python
from django.views.generic import ListView, DetailView
from .models import Post

class PostListView(ListView):
    model = Post
    template_name = "blog/list.html"
    context_object_name = "posts"
    queryset = Post.objects.filter(published=True)
    paginate_by = 10

class PostDetailView(DetailView):
    model = Post
    template_name = "blog/detail.html"
    # captures <int:pk> by default
```

Three lines per view, full-featured. Override `.get_queryset()` for custom logic.

--

## Generic views — `CreateView`, `UpdateView`, `DeleteView`

```python
from django.views.generic.edit import CreateView, UpdateView, DeleteView
from django.urls import reverse_lazy

class PostCreateView(CreateView):
    model = Post
    fields = ["title", "body"]            # auto-generates a form
    template_name = "blog/form.html"
    success_url = reverse_lazy("blog:list")

class PostUpdateView(UpdateView):
    model = Post
    fields = ["title", "body"]
    template_name = "blog/form.html"

class PostDeleteView(DeleteView):
    model = Post
    success_url = reverse_lazy("blog:list")
```

Form, validation, redirect — all generated. Override methods to customize.

---

## Mixins — composing behavior

```python
from django.contrib.auth.mixins import LoginRequiredMixin

class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    fields = ["title", "body"]
    login_url = "/accounts/login/"

class PostUpdateView(LoginRequiredMixin, UpdateView):
    ...
```

Common mixins: `LoginRequiredMixin`, `PermissionRequiredMixin`, `UserPassesTestMixin`. The base view goes **last** in the MRO.

---

## CBVs vs FBVs — when to pick

| Use FBVs when... | Use CBVs when... |
| --- | --- |
| The view does one specific thing | The view fits a generic pattern |
| Logic is procedural / hard to slice into class methods | You want to share behavior across many views |
| You want to read the code top-to-bottom | You're OK with the inheritance graph |
| The view is short | The view has many small variations |

Mix freely — same project, same urls.py.

---

## Decorators (FBV) vs mixins (CBV)

FBVs use decorators:

```python
from django.contrib.auth.decorators import login_required, permission_required

@login_required
@permission_required("blog.add_post")
def create_post(request):
    ...
```

CBVs use mixins (already shown). Same effect, different syntactic style.

---

## Async views

Django supports async views — useful when the view does I/O (HTTP calls, DB if your driver supports async).

```python
import httpx
from django.http import JsonResponse

async def fetch_status(request):
    async with httpx.AsyncClient() as client:
        r = await client.get("https://httpbin.org/get")
    return JsonResponse(r.json())
```

Run under an ASGI server (`uvicorn`, `daphne`). For most CRUD apps, the sync path is still simpler.

---

## Errors and HTTP status codes

```python
from django.http import HttpResponseBadRequest, HttpResponseNotFound, JsonResponse

return HttpResponseBadRequest("Missing 'q' parameter")     # 400
return HttpResponseNotFound()                              # 404
return JsonResponse({"error": "rate limited"}, status=429)
```

For consistent JSON error formats across an API, use **DRF**'s exception handlers (covered in DRF deck).

---

## Testing views

```python
# tests/test_views.py
from django.test import TestCase, Client

class PostViewTests(TestCase):
    def test_list_returns_200(self):
        c = Client()
        response = c.get("/blog/")
        assert response.status_code == 200

    def test_detail_404_for_missing(self):
        response = self.client.get("/blog/9999/")
        assert response.status_code == 404
```

With pytest + `pytest-django`, even cleaner — fixtures handle the test client.

---

## What's next

- **Templates** — rendering the response body
- **Auth** — `request.user`, login, sessions
- **Refactoring Django** — keeping views thin, models fat
