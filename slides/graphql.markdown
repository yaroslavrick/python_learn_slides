---
title: GraphQL
---

![GraphQL query selecting nested fields](/assets/images/topics/graphql.svg)
<!-- .element: class="title-illustration" -->

# GraphQL in Python

`Strawberry` and `Graphene-Django`.

---

## REST vs GraphQL

| | REST | GraphQL |
| --- | --- | --- |
| Endpoints | Many (`/posts/`, `/posts/1/comments/`) | One (`/graphql`) |
| Shape of response | Server decides | Client asks for what it wants |
| Versioning | URL or header (`/v1/`) | Schema evolution |
| Caching | HTTP-friendly | App-level only |
| Tooling | curl, Postman, OpenAPI | GraphiQL, persisted queries |

--

## When each wins

**GraphQL shines when:**

- One screen needs data from several resources
- Mobile clients want to minimize over-fetching on slow networks
- Frontend teams iterate fast on what fields they need

**REST shines when:**

- Resources are cacheable (HTTP caching is unbeatable)
- The API is small (3 endpoints, no schema overhead)
- A wide audience needs to read API docs (REST is the lingua franca)

---

## A GraphQL query

The client asks for **exactly the fields it wants**, including nested objects:

```graphql
query {
  post(id: 42) {
    title
    body
    author {
      name
      posts {
        title
      }
    }
  }
}
```

Press **↓** for the response.

--

## A GraphQL response

```json
{
  "data": {
    "post": {
      "title": "Hello",
      "body": "World",
      "author": {
        "name": "Alice",
        "posts": [
          {"title": "Hello"},
          {"title": "Goodbye"}
        ]
      }
    }
  }
}
```

Same shape as the query, top to bottom — that's the design.

---

## Two main libraries

| | Strawberry | Graphene |
| --- | --- | --- |
| Style | Type-hint-driven, code-first | Class-based, code-first |
| Async | First-class | Limited |
| Activity | Active, modern | Mature, slower releases |
| Django integration | `strawberry-django` | `graphene-django` |
| Type safety | Excellent (mypy/pyright friendly) | Manual |

--

## Which to pick?

For **new projects**: **Strawberry**. Type hints + first-class async + active development + a smaller surface area to learn.

For **existing projects on Graphene**: stay. Graphene is mature and well-documented. Migrating mid-project rarely pays for itself.

The rest of this deck uses Strawberry; Graphene appears at the end as a reference.

---

## Strawberry — install

```bash
uv add strawberry-graphql strawberry-graphql-django
```

```python
# settings.py
INSTALLED_APPS = [
    ...,
    "strawberry_django",
]
```

---

## Strawberry — defining types

```python
# blog/types.py
import strawberry
import strawberry_django
from .models import Post, Author

@strawberry_django.type(Author)
class AuthorType:
    id: int
    name: str
    posts: list["PostType"]

@strawberry_django.type(Post)
class PostType:
    id: int
    title: str
    body: str
    published: bool
    author: AuthorType
```

Type hints map to GraphQL types; relationships are auto-resolved.

---

## Strawberry — defining the Query

```python
# blog/schema.py
import strawberry
import strawberry_django
from .types import PostType, AuthorType

@strawberry.type
class Query:
    posts: list[PostType] = strawberry_django.field()
    post: PostType = strawberry_django.field()
    authors: list[AuthorType] = strawberry_django.field()

schema = strawberry.Schema(query=Query)
```

`strawberry_django.field()` auto-generates resolvers from the `@strawberry_django.type` definitions.

--

## Strawberry — wiring it up

```python
# urls.py
from django.urls import path
from strawberry.django.views import GraphQLView
from .schema import schema

urlpatterns = [
    path("graphql/", GraphQLView.as_view(schema=schema)),
]
```

Visit `/graphql/` in a browser — the **GraphiQL playground** is live: explorer, autocomplete, query history, schema docs.

---

## Strawberry — mutations

```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_post(self, title: str, body: str) -> PostType:
        return Post.objects.create(title=title, body=body)

    @strawberry.mutation
    def publish_post(self, id: strawberry.ID) -> PostType:
        post = Post.objects.get(pk=id)
        post.published = True
        post.save()
        return post

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

--

## Strawberry — calling a mutation

Client:

```graphql
mutation {
  createPost(title: "Hi", body: "World") {
    id
    title
  }
}
```

---

## Authentication and authorization

```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    def delete_post(self, info, id: strawberry.ID) -> bool:
        user = info.context.request.user
        if not user.is_authenticated:
            raise PermissionError("Must be logged in")
        post = Post.objects.get(pk=id)
        if post.author != user:
            raise PermissionError("Not your post")
        post.delete()
        return True
```

`info.context` exposes the Django request. For larger schemas, factor permission checks into helpers.

---

## The N+1 problem in GraphQL

```graphql
query {
  posts {
    title
    author { name }
  }
}
```

Naive resolution: one query for posts, then **one query per post** for the author. With 100 posts → 101 queries.

The fix is **DataLoader** — batches loads of related objects into one query.

---

## DataLoaders

```python
import strawberry
from strawberry.dataloader import DataLoader
from .models import Author

async def load_authors(keys: list[int]) -> list[Author]:
    authors = {a.id: a for a in await Author.objects.filter(id__in=keys).aall()}
    return [authors[k] for k in keys]

author_loader = DataLoader(load_fn=load_authors)

# In a resolver:
async def resolver_post_author(post):
    return await author_loader.load(post.author_id)
```

`strawberry-django` integrates this; `select_related`/`prefetch_related` covers most cases without DataLoader.

---

## Subscriptions (real-time)

```python
@strawberry.type
class Subscription:
    @strawberry.subscription
    async def post_published(self) -> AsyncGenerator[PostType, None]:
        async for post in event_bus.subscribe("post.published"):
            yield post
```

Requires:

- An ASGI server (uvicorn, daphne)
- A real-time channel (Redis pub/sub, channels)

For most CRUD APIs, queries + mutations are enough; reach for subscriptions only when push beats poll.

---

## Graphene-Django — types

```bash
uv add graphene-django
```

```python
# blog/schema.py
import graphene
from graphene_django.types import DjangoObjectType
from .models import Post

class PostType(DjangoObjectType):
    class Meta:
        model = Post
        fields = ("id", "title", "body", "author")
```

`DjangoObjectType` reads your Django model and exposes the listed fields.

--

## Graphene-Django — query

```python
class Query(graphene.ObjectType):
    all_posts = graphene.List(PostType)
    post = graphene.Field(PostType, id=graphene.Int())

    def resolve_all_posts(self, info):
        return Post.objects.all()

    def resolve_post(self, info, id):
        return Post.objects.get(pk=id)

schema = graphene.Schema(query=Query)
```

Class-based, more verbose than Strawberry, but well-documented and stable.

---

## Tooling

| Tool | What it does |
| --- | --- |
| **GraphiQL** | Browser playground for queries (built into both libraries) |
| **Apollo Studio** | Schema management, query plans, performance |
| **graphql-codegen** | Generate TypeScript types from your schema for the frontend |
| **persistgraphql / Apollo persisted queries** | Whitelist queries in production (security + caching) |

---

## When GraphQL pays off

- **Frontend has many "screens"** — each fetching different combinations of fields
- **Mobile apps** — minimize over-fetching on slow networks
- **Aggregator APIs** — one endpoint pulling from multiple services
- **Schema-first contract** — the schema is the API spec, code-generated types on both sides

---

## When REST is still better

- **Tiny APIs** — 3 endpoints, no client-side complexity → REST is shorter
- **Heavy caching** — REST + HTTP caching is unbeatable
- **Public API** — REST is more familiar; GraphQL adds learning curve
- **Per-resource auth model** — easier to reason about per URL

You don't have to pick one — REST and GraphQL can coexist on the same Django project.

---

## What's next

- **FastAPI** — async-first APIs without Django
- **Async & concurrency** — when GraphQL subscriptions and async views matter
