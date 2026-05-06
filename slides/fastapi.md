---
title: FastAPI
---

![Type-hinted handler producing JSON](/assets/images/topics/fastapi.svg)
<!-- .element: class="title-illustration" -->

# FastAPI

Modern, async, type-hint-driven Python APIs.

---

## Why FastAPI?

- **Type hints drive everything** — request validation, response serialization, OpenAPI docs
- **Async-first** — `async def` views, async DB drivers, async HTTP clients
- **Pydantic** for data validation — fast, strict, friendly error messages
- **Auto-generated docs** at `/docs` and `/redoc` — Swagger UI, free
- **Tiny** — one process, one file is a complete app

For JSON APIs, FastAPI is often simpler than Django + DRF and faster than both.

---

## Start a project

```bash
uv init my-api
cd my-api
uv add "fastapi[standard]"
```

`fastapi[standard]` pulls in the runtime essentials: `uvicorn`, `pydantic`, `httpx`, the `fastapi` CLI.

After this, you have:

```
my-api/
├── pyproject.toml
├── uv.lock
├── .python-version
├── README.md
└── main.py            # uv init's stub — we'll replace it
```

---

## Hello, FastAPI

Replace `main.py` with:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"hello": "world"}
```

Press **↓** to run it.

--

## Run it

```bash
uv run fastapi dev main.py
# INFO     Will watch for changes in: ['/Users/me/my-api']
# INFO     Uvicorn running on http://127.0.0.1:8000
```

Auto-reloads on file changes. Visit:

- `/` → `{"hello": "world"}`
- `/docs` → **Swagger UI** (interactive)
- `/redoc` → **ReDoc UI** (read-only docs)
- `/openapi.json` → the schema, machine-readable

`fastapi dev` is the dev-mode shortcut; for production use `uvicorn main:app --workers 4`.

---

## Path parameters

```python
@app.get("/posts/{post_id}")
def get_post(post_id: int):
    return {"id": post_id}
```

The type hint `int` does the work:

- `GET /posts/42` → `{"id": 42}`
- `GET /posts/abc` → `422 Unprocessable Entity` with a clear error

No manual parsing, no `try/except` around `int(...)`.

---

## Query parameters

```python
@app.get("/posts/")
def list_posts(limit: int = 10, offset: int = 0, q: str | None = None):
    return {"limit": limit, "offset": offset, "q": q}
```

- `GET /posts/?limit=5&q=django` → `{"limit": 5, "offset": 0, "q": "django"}`
- `GET /posts/` → defaults applied
- `GET /posts/?limit=abc` → 422

Defaults make params optional; `int | None = None` makes them nullable too.

--

## Grouping params — the Rails-style pattern

For endpoints with several related params (or shared across multiple handlers), group them into a Pydantic model — the FastAPI equivalent of Rails **Strong Parameters**.

```python
from typing import Annotated
from pydantic import BaseModel, Field
from fastapi import Depends

class PostIndexParams(BaseModel):
    limit:  int = Field(10, ge=1, le=100)      # validation in one place
    offset: int = Field(0,  ge=0)
    q:      str | None = None

@app.get("/posts/")
def list_posts(params: Annotated[PostIndexParams, Depends()]):
    return params.model_dump()
```

Same URLs, same 422s, same OpenAPI docs — but the params are a **named, reusable type**.

--

## Why group params into a model?

- **Reusable** — share `PostIndexParams` across `list_posts`, `count_posts`, an admin export, ...
- **Validated in one place** — `Field(ge=1, le=100)` is the contract; no per-handler `if limit > 100` checks
- **Strict by default** — `model_config = ConfigDict(extra="forbid")` rejects unknown query params (the "permit" part of Rails Strong Params)
- **Documented** — OpenAPI / Swagger UI shows the model as a reusable schema, not a flat list of inline params

--

## Strict params — example

```python
from pydantic import BaseModel, ConfigDict, Field

class PostIndexParams(BaseModel):
    model_config = ConfigDict(extra="forbid")
    limit:  int = Field(10, ge=1, le=100)
    offset: int = Field(0, ge=0)

# GET /posts/?limit=10            →  200
# GET /posts/?limit=10&evil=1     →  422   ('evil' rejected)
# GET /posts/?limit=999           →  422   (le=100 violated)
```

For request bodies (POST / PUT / PATCH), the same Pydantic-model pattern is the **default** — covered on the next slides.

---

## Request bodies — Pydantic models

```python
from pydantic import BaseModel

class PostIn(BaseModel):
    title: str
    body: str
    tags: list[str] = []

@app.post("/posts/")
def create_post(post: PostIn):
    return {"id": 1, **post.model_dump()}
```

```bash
curl -X POST localhost:8000/posts/ \
  -H "Content-Type: application/json" \
  -d '{"title":"Hi","body":"World","tags":["python"]}'
# {"id": 1, "title": "Hi", "body": "World", "tags": ["python"]}
```

Validation, parsing, JSON Schema generation — all from one class.

---

## Response models

```python
class PostOut(BaseModel):
    id: int
    title: str
    body: str
    created_at: datetime

@app.post("/posts/", response_model=PostOut, status_code=201)
def create_post(post: PostIn) -> PostOut:
    saved = save_post(post)
    return saved          # FastAPI strips fields not in PostOut
```

`response_model` shapes the **outgoing** JSON — even if the function returns a richer object, only the declared fields are serialized.

---

## Status codes and exceptions

```python
from fastapi import HTTPException

@app.get("/posts/{id}", response_model=PostOut)
def get_post(id: int):
    post = Post.get(id)
    if post is None:
        raise HTTPException(status_code=404, detail="Post not found")
    return post

@app.post("/posts/", status_code=201)
def create(post: PostIn):
    ...
```

`HTTPException` becomes a JSON error response with the right status. For multi-error validation flows, use Pydantic — it does the heavy lifting.

---

## Dependency injection

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/posts/")
def list_posts(db = Depends(get_db)):
    return db.query(Post).all()
```

`Depends` runs the dependency, injects its return value. `yield` makes it a generator-based dependency with cleanup — for DBs, files, locks.

--

## Dependency injection — composing

```python
def get_db():
    ...

def get_user(token: str = Header(), db = Depends(get_db)):
    user = decode_token(token, db)
    if user is None:
        raise HTTPException(401)
    return user

@app.get("/me/")
def me(user = Depends(get_user)):       # ← user; db injected transitively
    return user
```

Dependencies depend on dependencies. Same `Depends(get_db)` call in different handlers reuses the result within one request.

---

## Pydantic — validation

```python
from pydantic import BaseModel, EmailStr, Field, field_validator

class UserIn(BaseModel):
    email: EmailStr
    name: str = Field(min_length=2, max_length=80)
    age: int = Field(ge=13, le=120)

    @field_validator("name")
    @classmethod
    def no_digits(cls, v):
        if any(c.isdigit() for c in v):
            raise ValueError("name cannot contain digits")
        return v
```

Bad input → automatic 422 with a per-field error array.

--

## Pydantic — nested models

```python
class Address(BaseModel):
    line1: str
    city: str
    zip: str

class UserIn(BaseModel):
    name: str
    email: EmailStr
    addresses: list[Address] = []
    profile: dict[str, str] = {}
```

```bash
curl -X POST localhost:8000/users/ -d '{
  "name":"Alice","email":"a@x.com",
  "addresses":[{"line1":"1 St","city":"NYC","zip":"10001"}]
}'
```

Validation traverses nested structures. JSON in → typed objects out.

---

## Async views

```python
import httpx

@app.get("/proxy")
async def proxy(url: str):
    async with httpx.AsyncClient() as client:
        r = await client.get(url)
    return r.json()
```

Async views are the right call when the handler does I/O — databases (with async drivers), HTTP, message brokers. CPU-bound work belongs in a thread or process pool, not the event loop.

---

## APIRouter — splitting the app

```python
# blog/router.py
from fastapi import APIRouter

router = APIRouter(prefix="/posts", tags=["posts"])

@router.get("/")
def list_posts(): ...

@router.get("/{id}")
def get_post(id: int): ...
```

```python
# main.py
from fastapi import FastAPI
from blog import router as blog_router

app = FastAPI()
app.include_router(blog_router.router)
```

Like Django's `include()`, but lives directly in your Python module.

---

## Authentication — OAuth2 + JWT

```python
from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer

oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/token")

def current_user(token: str = Depends(oauth2)):
    user = decode_jwt(token)
    if not user:
        raise HTTPException(401)
    return user

@app.get("/me/")
def me(user = Depends(current_user)):
    return user
```

Token endpoint returns a JWT; protected endpoints receive `token` from the `Authorization: Bearer ...` header.

---

## Background tasks

```python
from fastapi import BackgroundTasks

def send_welcome(email: str):
    smtplib.SMTP(...).send(...)

@app.post("/users/")
def create_user(user: UserIn, bt: BackgroundTasks):
    saved = save(user)
    bt.add_task(send_welcome, saved.email)
    return saved
```

`BackgroundTasks` runs **after the response is sent**, in the same process. For heavier or retryable work, reach for **Celery** / **rq** / **dramatiq**.

---

## CORS, gzip, middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

Other built-ins: `GZipMiddleware`, `TrustedHostMiddleware`, `HTTPSRedirectMiddleware`. Custom middleware is a function decorated with `@app.middleware("http")`.

---

## OpenAPI — for free

FastAPI generates an OpenAPI 3.x schema from your type hints, Pydantic models, and route metadata.

- `/openapi.json` — the schema
- `/docs` — Swagger UI (interactive)
- `/redoc` — ReDoc (read-only docs)

Use the schema with code generators (`openapi-generator`, `openapi-python-client`) to ship typed clients in TS, Python, Go, ...

---

## Testing

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_post():
    r = client.post("/posts/", json={"title": "Hi", "body": "World"})
    assert r.status_code == 201
    assert r.json()["title"] == "Hi"
```

`TestClient` runs your app in-process — no actual network. For async-only setups, use `httpx.AsyncClient(app=app, base_url="http://test")`.

---

## FastAPI vs Django

| | FastAPI | Django |
| --- | --- | --- |
| Best for | JSON APIs, async, microservices | Full-stack apps with admin, templates, auth UI |
| Routing style | Decorators on functions | `urls.py` → views |
| ORM | Bring your own (SQLAlchemy, SQLModel, Tortoise) | Built-in |
| Templates | Jinja2 if you need it | DTL built-in |
| Admin | Build it yourself | Free, mature |
| Async | First-class | Available, sync still primary |

They don't compete on every dimension. Pick by the shape of the project.

---

## What's next

- **WSGI / ASGI** — the protocols underneath
- **Async & concurrency** — when async pays off, when it doesn't
