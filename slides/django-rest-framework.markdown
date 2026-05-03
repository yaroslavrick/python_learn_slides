---
title: Django REST Framework
---

![REST endpoints producing JSON](/assets/images/topics/django-rest-framework.svg)
<!-- .element: class="title-illustration" -->

# Django REST Framework

Serializers, viewsets, routers, auth, throttling, pagination.

---

## Why DRF?

For JSON APIs over Django models. DRF gives you:

- **Serializers** — JSON ↔ model conversion + validation
- **ViewSets** — list / retrieve / create / update / delete generated from one class
- **Routers** — auto-generate URL patterns
- **Browsable API** — explore your API in a web UI for free
- **Auth, permissions, throttling** — pluggable, sane defaults
- **Pagination, filtering, search** — built-in

For a UI-driven Django app, the framework's templates suffice. For a JSON API, DRF is the obvious next step.

---

## Install

```bash
uv add djangorestframework
```

```python
# settings.py
INSTALLED_APPS = [
    ...,
    "rest_framework",
]
```

That's it for a basic setup. `rest_framework.urls` is optional (browsable-API login).

---

## Serializer — model ↔ JSON

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ["id", "title", "body", "author", "created_at"]
        read_only_fields = ["created_at"]
```

Translates a `Post` to JSON and back, with field-level validation.

---

## Using a serializer

```python
post = Post.objects.get(pk=1)
PostSerializer(post).data
# {'id': 1, 'title': 'Hello', 'body': 'World', 'author': 7, 'created_at': '...'}

# Deserialize
data = {"title": "Hi", "body": "World", "author": 7}
ser = PostSerializer(data=data)
ser.is_valid()                  # True
ser.save()                      # creates a Post

ser = PostSerializer(data={"title": ""})
ser.is_valid()                  # False
ser.errors                      # {'title': ['This field may not be blank.']}
```

`is_valid(raise_exception=True)` raises `ValidationError` (DRF returns 400 automatically).

---

## Custom validation

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ["title", "body"]

    def validate_title(self, value):
        if len(value) < 5:
            raise serializers.ValidationError("Title too short.")
        return value

    def validate(self, attrs):
        # Cross-field validation
        if attrs["title"].lower() in attrs["body"].lower():
            raise serializers.ValidationError("Body must add info.")
        return attrs
```

`validate_<field>` runs per-field; `validate(attrs)` runs after all per-field passes.

---

## Function views with `@api_view`

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .serializers import PostSerializer
from .models import Post

@api_view(["GET", "POST"])
def post_list(request):
    if request.method == "GET":
        posts = Post.objects.all()
        return Response(PostSerializer(posts, many=True).data)

    ser = PostSerializer(data=request.data)
    ser.is_valid(raise_exception=True)
    ser.save()
    return Response(ser.data, status=201)
```

`request.data` is the parsed body (JSON, form, multipart — DRF figures it out).
`Response` returns content negotiated to JSON / browsable HTML.

---

## ViewSets — DRY for CRUD

```python
from rest_framework import viewsets
from .models import Post
from .serializers import PostSerializer

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

Six lines. You now have:

- `GET /posts/` — list
- `POST /posts/` — create
- `GET /posts/{id}/` — retrieve
- `PUT /posts/{id}/`, `PATCH /posts/{id}/` — update
- `DELETE /posts/{id}/` — destroy

---

## Routers — wiring viewsets

```python
# blog/api/urls.py
from rest_framework.routers import DefaultRouter
from .views import PostViewSet

router = DefaultRouter()
router.register(r"posts", PostViewSet, basename="post")

urlpatterns = router.urls
```

```python
# mysite/urls.py
urlpatterns = [
    path("api/", include("blog.api.urls")),
]
```

`DefaultRouter` also wires up the API root (`/api/`) and a format suffix (`/posts/.json`).

---

## Customizing a ViewSet

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    filterset_fields = ["author", "published"]
    search_fields = ["title", "body"]
    ordering_fields = ["created_at", "title"]

    def get_queryset(self):
        qs = super().get_queryset()
        if self.action == "list":
            qs = qs.filter(published=True)
        return qs

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

`get_queryset` and `perform_create` are the most-overridden hooks.

---

## Custom actions on a ViewSet

```python
from rest_framework.decorators import action
from rest_framework.response import Response

class PostViewSet(viewsets.ModelViewSet):
    ...

    @action(detail=True, methods=["post"])
    def publish(self, request, pk=None):
        post = self.get_object()
        post.published = True
        post.save()
        return Response(self.get_serializer(post).data)

    @action(detail=False)
    def featured(self, request):
        qs = self.get_queryset().filter(featured=True)
        return Response(self.get_serializer(qs, many=True).data)
```

Routes auto-added: `POST /posts/{id}/publish/`, `GET /posts/featured/`.

---

## Authentication

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
}
```

`SessionAuthentication` — for browser clients (cookie + CSRF).
`TokenAuthentication` — for non-browser clients (mobile, server-to-server).

---

## JWT auth — `djangorestframework-simplejwt`

```bash
uv add djangorestframework-simplejwt
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view()),
    path("api/token/refresh/", TokenRefreshView.as_view()),
]
```

Client posts username/password to `/api/token/` → gets access + refresh tokens.

---

## Permissions

```python
from rest_framework import permissions

class IsAuthorOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user

class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticated, IsAuthorOrReadOnly]
```

Built-ins: `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions`.

---

## Pagination

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 25,
}
```

Response shape:

```json
{
  "count": 142,
  "next": "https://api.example.com/posts/?page=2",
  "previous": null,
  "results": [ ... ]
}
```

Other built-in classes: `LimitOffsetPagination`, `CursorPagination` (for unstable orderings on huge tables).

---

## Throttling

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/day",
        "user": "1000/day",
    },
}
```

Per-view overrides:

```python
class LoginView(APIView):
    throttle_classes = [AnonRateThrottle]
    throttle_scope = "login"
```

Throttle backed by Django's cache — for production, point cache at Redis.

---

## Filtering and search

```bash
uv add django-filter
```

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
}
```

```python
class PostViewSet(viewsets.ModelViewSet):
    filterset_fields = ["author", "published"]
    search_fields = ["title", "body"]
    ordering_fields = ["created_at", "title"]
```

`/posts/?author=7&search=django&ordering=-created_at` → filtered, searched, ordered.

---

## Browsable API

By default, DRF renders a usable HTML UI for browsers — read endpoints, fill in POST forms, see the JSON schema. Disable in production:

```python
REST_FRAMEWORK = {
    "DEFAULT_RENDERER_CLASSES": [
        "rest_framework.renderers.JSONRenderer",
        # remove BrowsableAPIRenderer in prod
    ],
}
```

For a polished schema UI, add `drf-spectacular` (OpenAPI 3) and serve the spec with Swagger UI.

---

## Testing DRF endpoints

```python
# tests/test_api.py
from rest_framework.test import APIClient
from rest_framework import status

def test_create_post(db):
    user = UserFactory()
    client = APIClient()
    client.force_authenticate(user)
    r = client.post("/api/posts/", {"title": "Hi", "body": "World"})
    assert r.status_code == status.HTTP_201_CREATED
    assert r.json()["title"] == "Hi"
```

`APIClient.force_authenticate` skips the auth flow for tests.

---

## When DRF is too much — alternatives

- **Django Ninja** — FastAPI-style decorators on Django; great for type-hint-driven APIs.
- **FastAPI** + a separate ORM (SQLAlchemy / SQLModel) — when you want async / non-Django-ORM.
- **Plain Django views with `JsonResponse`** — for tiny APIs (one or two endpoints).

DRF wins where the project already has Django models and many CRUD-shaped endpoints.

---

## What's next

- **GraphQL** — alternative query model with one endpoint
- **FastAPI** — when you want async-first, type-hint-driven
