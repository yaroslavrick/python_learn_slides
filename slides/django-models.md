---
title: Django models
---

![Django models with relationships](/assets/images/topics/django-models.svg)
<!-- .element: class="title-illustration" -->

# Django models

The ORM, migrations, querysets, relationships.

---

## What models are

A **model** is a Python class that maps to a database table.

```python
# blog/models.py
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    published = models.BooleanField(default=False)
```

One class → one table → instances are rows.

---

## Common field types

| Field | Maps to |
| --- | --- |
| `CharField(max_length=...)` | `varchar` |
| `TextField` | `text` |
| `IntegerField` / `BigIntegerField` | `integer` / `bigint` |
| `FloatField` / `DecimalField` | `float` / `numeric` |
| `BooleanField` | `boolean` |
| `DateField` / `DateTimeField` | `date` / `timestamptz` |
| `EmailField`, `URLField`, `SlugField` | `varchar` (with validation) |
| `JSONField` | `jsonb` |
| `UUIDField` | `uuid` |
| `FileField` / `ImageField` | `varchar` + filesystem |

---

## Field options

```python
class Post(models.Model):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)
    body = models.TextField(blank=True)               # form may be empty
    score = models.IntegerField(null=True)            # DB may be NULL
    tags = models.JSONField(default=list)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

- `null=True` — DB column nullable
- `blank=True` — forms allow empty (different concern!)
- `default=...` — default at the model level
- `unique=True` — DB unique constraint
- `db_index=True` — add an index

---

## Migrations

```bash
uv run python manage.py makemigrations
# Migrations for 'blog':
#   blog/migrations/0001_initial.py
#     - Create model Post

uv run python manage.py migrate
# Operations to perform:
#   Apply all migrations: admin, auth, blog, ...
#   Applying blog.0001_initial... OK
```

Migrations are versioned Python files describing schema changes. **Commit them**.

--

## Migrations — workflow

```bash
# After editing models.py:
uv run python manage.py makemigrations blog --name=add_published_field
# blog/migrations/0002_add_published_field.py

# Inspect the SQL it'll run:
uv run python manage.py sqlmigrate blog 0002

# Apply
uv run python manage.py migrate

# Roll back to a prior migration
uv run python manage.py migrate blog 0001
```

Squash old migrations once a release is stable: `python manage.py squashmigrations`.

---

## Creating and saving

```python
post = Post(title="Hello", body="World")
post.save()                              # → INSERT
post.id                                  # 1

# One-shot create
p = Post.objects.create(title="Hello", body="World")

# Update
p.title = "Hi"
p.save()                                 # → UPDATE

# Delete
p.delete()                               # → DELETE WHERE id=1
```

`.objects` is the default **manager** — every model has one.

---

## Queries

```python
Post.objects.all()                       # → SELECT * FROM blog_post
Post.objects.count()                     # 42
Post.objects.first()                     # <Post: Hello>
Post.objects.last()
Post.objects.get(id=1)                   # raises if not found / multiple

# Filter
Post.objects.filter(published=True)
Post.objects.filter(title__icontains="django")
Post.objects.filter(created_at__gte=yesterday)

# Exclude
Post.objects.exclude(score__lt=0)

# Ordering
Post.objects.order_by("-created_at")     # DESC
```

Each call returns a **QuerySet** — lazy, chainable, evaluated when iterated.

--

## Field lookups

`__lookup` on field names:

```python
Post.objects.filter(title__exact="Hi")          # =
Post.objects.filter(title__iexact="hi")         # case-insensitive =
Post.objects.filter(title__contains="ng")       # LIKE %ng%
Post.objects.filter(title__icontains="NG")      # case-insensitive LIKE
Post.objects.filter(title__startswith="Dj")
Post.objects.filter(score__gt=10)               # >
Post.objects.filter(score__gte=10)              # >=
Post.objects.filter(score__in=[1, 2, 3])        # IN
Post.objects.filter(score__isnull=True)         # IS NULL
Post.objects.filter(created_at__date=today)
Post.objects.filter(tags__contains=["python"])  # JSONField
```

--

## Q objects — OR / NOT

`filter(...)` calls AND together. For OR or NOT, use `Q`:

```python
from django.db.models import Q

Post.objects.filter(Q(published=True) | Q(score__gte=100))
# WHERE published=true OR score >= 100

Post.objects.filter(~Q(title=""))
# WHERE NOT title = ''

# Combine
Post.objects.filter(
    Q(category="news") | Q(category="blog"),
    published=True,                               # AND'd with the Q
)
```

---

## Relationships

```python
class Author(models.Model):
    name = models.CharField(max_length=100)

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE,
                               related_name="posts")
    tags = models.ManyToManyField("Tag", related_name="posts")

class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
```

- `ForeignKey` — many-to-one (each post has one author)
- `ManyToManyField` — many-to-many (posts ↔ tags)
- `OneToOneField` — one-to-one (Profile ↔ User)

--

## Foreign keys — using them

```python
alice = Author.objects.create(name="Alice")
post = Post.objects.create(title="Hello", author=alice)

# Forward (post → author)
post.author                           # <Author: Alice>
post.author.name                      # 'Alice'

# Reverse (author → posts)  — uses related_name="posts"
alice.posts.all()                     # <QuerySet [<Post: Hello>]>
alice.posts.filter(published=True)
alice.posts.count()                   # 1
```

`on_delete=` says what happens when the referenced row is deleted. Common: `CASCADE`, `PROTECT`, `SET_NULL`, `SET_DEFAULT`.

--

## Many-to-many — using them

```python
post = Post.objects.create(title="Django models")
python = Tag.objects.create(name="python")
django = Tag.objects.create(name="django")

post.tags.add(python, django)         # → INSERT 2 rows in m2m table
post.tags.all()                       # <QuerySet [<Tag: python>, <Tag: django>]>
post.tags.remove(python)              # → DELETE row
post.tags.clear()                     # → DELETE all rows for this post

# Reverse (tag → posts)
django.posts.all()                    # all posts tagged 'django'
```

--

## Through models

When the m2m relationship needs **its own data** (when, by whom, ...):

```python
class Membership(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    group = models.ForeignKey(Group, on_delete=models.CASCADE)
    joined_at = models.DateTimeField(auto_now_add=True)
    role = models.CharField(max_length=20, default="member")

class Group(models.Model):
    name = models.CharField(max_length=100)
    members = models.ManyToManyField(User, through=Membership)
```

`group.members.all()` works as before, but you also have `Membership.objects.filter(...)` for the relationship-level data.

---

## Performance — N+1 queries

```python
# Bad: one query for posts, then one per post for the author
for post in Post.objects.all():
    print(post.title, post.author.name)
# 1 + N queries
```

The `.author` access for each post triggers a separate `SELECT`.

--

## Performance — `select_related`

For **forward** ForeignKey / OneToOne — does a `JOIN`:

```python
for post in Post.objects.select_related("author"):
    print(post.title, post.author.name)
# 1 query total
```

```sql
SELECT post.*, author.*
  FROM blog_post
  JOIN blog_author ON post.author_id = author.id
```

Use for `ForeignKey` and `OneToOneField`.

--

## Performance — `prefetch_related`

For **reverse** ForeignKey and **ManyToMany** — second query, joined in Python:

```python
for author in Author.objects.prefetch_related("posts"):
    for post in author.posts.all():
        print(author.name, post.title)
# 2 queries total: SELECT authors, then SELECT posts WHERE author_id IN (...)
```

Combine both:

```python
Post.objects.select_related("author").prefetch_related("tags")
```

---

## Aggregates and annotations

```python
from django.db.models import Count, Avg, Max, Sum

Post.objects.aggregate(total=Count("id"))
# {'total': 42}

Author.objects.annotate(post_count=Count("posts"))
# Each Author now has .post_count

Author.objects.annotate(post_count=Count("posts")).filter(post_count__gte=5)
# Authors with 5+ posts
```

`aggregate` returns a dict; `annotate` adds a field per row.

---

## Custom managers

Encapsulate common filters once:

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    title = models.CharField(max_length=200)
    published = models.BooleanField(default=False)

    objects = models.Manager()       # default
    published_only = PublishedManager()

Post.published_only.all()            # only published
Post.published_only.filter(title__icontains="django")
```

For chainable custom methods, prefer **custom QuerySets** (covered in Refactoring Django).

---

## The Django shell

```bash
uv run python manage.py shell
```

```python
>>> from blog.models import Post, Author
>>> alice = Author.objects.create(name="Alice")
>>> Post.objects.create(title="Hi", author=alice)
<Post: Hi>
>>> Post.objects.count()
1
```

`shell_plus` (from `django-extensions`) auto-imports your models — worth installing.

---

## What's next

- **URLs** — routing requests to views
- **Views** — function- and class-based
- **Refactoring Django** — service layer, custom querysets
