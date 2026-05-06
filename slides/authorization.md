---
title: Authorization
---

![Shield with permission checks](/assets/images/topics/authorization.svg)
<!-- .element: class="title-illustration" -->

# Authorization

Permissions, groups, object-level access.

---

## Authentication vs authorization

| | What it answers |
| --- | --- |
| **Authentication** | "Who is this user?" |
| **Authorization** | "What is this user allowed to do?" |

Authentication checks identity (covered in **Django auth**). Authorization checks privilege.

---

## Built-in permissions

When you `migrate`, Django auto-creates four permissions per model:

```
blog | post | Can add post
blog | post | Can change post
blog | post | Can delete post
blog | post | Can view post
```

Use them in views:

```python
from django.contrib.auth.decorators import permission_required

@permission_required("blog.add_post")
def create_post(request):
    ...

@permission_required("blog.change_post", raise_exception=True)
def edit_post(request, id):
    ...
```

`raise_exception=True` returns 403 instead of redirecting to login.

---

## Custom permissions

```python
class Post(models.Model):
    ...

    class Meta:
        permissions = [
            ("publish_post", "Can publish a post"),
            ("feature_post", "Can mark a post as featured"),
        ]
```

After `makemigrations` + `migrate`:

```python
@permission_required("blog.publish_post")
def publish(request, id):
    ...
```

Custom permissions live alongside the auto-generated ones in `auth_permission`.

---

## Checking permissions in code

```python
def my_view(request):
    request.user.has_perm("blog.add_post")           # bool
    request.user.has_perms(["blog.add_post", "blog.publish_post"])
    request.user.get_all_permissions()               # set of 'app.codename'
```

In templates:

```django
{% raw %}{% if perms.blog.add_post %}
  <a href="{% url 'blog:create' %}">New post</a>
{% endif %}{% endraw %}
```

`perms` is auto-injected via the `auth` context processor.

---

## Groups

A group is a named bundle of permissions:

```python
from django.contrib.auth.models import Group, Permission

editors = Group.objects.create(name="Editors")
editors.permissions.add(
    Permission.objects.get(codename="add_post"),
    Permission.objects.get(codename="change_post"),
    Permission.objects.get(codename="publish_post"),
)

alice.groups.add(editors)
alice.has_perm("blog.publish_post")   # True (inherited from group)
```

Manage groups in the admin (`/admin/auth/group/`). Avoid assigning permissions directly to users — assign to groups, put users in groups.

---

## CBV — `PermissionRequiredMixin`

```python
from django.contrib.auth.mixins import PermissionRequiredMixin
from django.views.generic.edit import CreateView

class PostCreateView(PermissionRequiredMixin, CreateView):
    permission_required = "blog.add_post"
    raise_exception = True              # 403 instead of redirect
    model = Post
    fields = ["title", "body"]
```

Multiple permissions:

```python
class PostPublishView(PermissionRequiredMixin, View):
    permission_required = ("blog.change_post", "blog.publish_post")
```

---

## Object-level: `UserPassesTestMixin`

For "only the post's author can edit it" type rules:

```python
from django.contrib.auth.mixins import UserPassesTestMixin

class PostUpdateView(UserPassesTestMixin, UpdateView):
    model = Post
    fields = ["title", "body"]

    def test_func(self):
        return self.get_object().author == self.request.user
```

Returns `403 Forbidden` (or redirects to login) when `test_func` is `False`.

---

## Object-level — `django-guardian`

The built-in permission system is **model-level**: "can edit any post". For "can edit *this* post":

```bash
uv add django-guardian
```

```python
# settings.py
INSTALLED_APPS += ["guardian"]
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "guardian.backends.ObjectPermissionBackend",
]
```

```python
from guardian.shortcuts import assign_perm, get_objects_for_user

assign_perm("change_post", alice, post)         # alice can change THIS post
alice.has_perm("change_post", post)             # True
get_objects_for_user(alice, "blog.change_post") # all posts alice can change
```

Use Guardian when ownership / sharing rules are non-trivial.

---

## Authorization in DRF

DRF has its own permission system that wraps Django's:

```python
from rest_framework import permissions

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]
```

Built-ins: `AllowAny`, `IsAuthenticated`, `IsAdminUser`, `IsAuthenticatedOrReadOnly`, `DjangoModelPermissions`. Custom classes implement `.has_permission()` and `.has_object_permission()`. We'll cover this in the **DRF** deck.

---

## A custom permission class (DRF)

```python
from rest_framework import permissions

class IsAuthorOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.author == request.user

# Usage
class PostViewSet(viewsets.ModelViewSet):
    permission_classes = [permissions.IsAuthenticated, IsAuthorOrReadOnly]
```

`SAFE_METHODS` is `("GET", "HEAD", "OPTIONS")`. Read for everyone authenticated, write only for the author.

---

## Roles vs permissions

Django doesn't ship roles — roles are usually just **groups** with documented intent.

| Convention | Implementation |
| --- | --- |
| "Editor" role | a `Group` named "Editors" with relevant permissions |
| "Admin" role | `is_staff=True` (or a Group, depending on what "admin" means) |
| "Superuser" | `is_superuser=True` — bypasses all permission checks |

For complex RBAC: `django-rules`, `django-role-permissions`, or model your own `Role` if the domain is rich.

---

## Auditing access

For sensitive actions, log who did what:

```python
import logging
logger = logging.getLogger(__name__)

class PostDeleteView(PermissionRequiredMixin, DeleteView):
    permission_required = "blog.delete_post"

    def delete(self, request, *args, **kwargs):
        post = self.get_object()
        logger.warning("user=%s deleted post=%s", request.user.id, post.id)
        return super().delete(request, *args, **kwargs)
```

For full audit trails: `django-auditlog`, `django-simple-history`, or write your own append-only log model.

---

## Don't reinvent the basics

- **Don't** roll your own decorators for "is admin" checks — use Django's
- **Don't** scatter `if request.user.has_perm(...)` across views — use `@permission_required` / mixins
- **Do** put rules in tests so you can refactor with confidence

---

## What's next

- **Django apps** — packaging features into reusable apps
- **DRF** — permissions for API endpoints
- **Refactoring Django** — keeping authorization out of views
