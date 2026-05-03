---
title: Metaprogramming
---

![Nested decorators wrapping a function](/assets/images/topics/metaprogramming.svg)
<!-- .element: class="title-illustration" -->

# Metaprogramming

Code that writes, inspects, or alters other code.

---

## What we'll cover

- Decorators
- Descriptors
- `__getattr__` and friends
- Dynamic class creation
- Metaclasses
- `__init_subclass__` (the modern alternative)
- Real-world examples

---

## Functions are objects

```python
def greet(name):
    return f"Hi, {name}"

f = greet                   # rebind a name
f("Alice")                  # 'Hi, Alice'

def call_it(fn, x):         # pass them around
    return fn(x)

call_it(greet, "Bob")       # 'Hi, Bob'
```

This is the foundation of every metaprogramming trick in Python.

---

## Decorators: the idea

A decorator wraps a function in another function.

```python
def shout(fn):
    def wrapper(*args, **kwargs):
        result = fn(*args, **kwargs)
        return result.upper()
    return wrapper

def greet(name):
    return f"Hi, {name}"

greet = shout(greet)
greet("Alice")              # 'HI, ALICE'
```

The `@` syntax is sugar for that rebind.

---

## The @ syntax

```python
@shout
def greet(name):
    return f"Hi, {name}"
```

is exactly equivalent to:

```python
def greet(name):
    return f"Hi, {name}"
greet = shout(greet)
```

Decorators run **at definition time**.

---

## Preserving metadata

```python
import functools

def shout(fn):
    @functools.wraps(fn)        # ← carry over name, doc, signature
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs).upper()
    return wrapper

@shout
def greet(name):
    """Say hi."""
    return f"Hi, {name}"

greet.__name__              # 'greet'
greet.__doc__               # 'Say hi.'
```

Always use `functools.wraps`.

---

## Decorators with arguments

A decorator factory: a callable that returns a decorator.

```python
def repeat(times):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = fn(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def ping(): print("ping")

ping()                      # prints "ping" three times
```

--

## Real-world decorators — `@cache` / `@lru_cache`

`functools` ships memoization out of the box.

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

fib(100)                    # 354224848179261915075   (instant)
fib.cache_info()
# CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)
fib.cache_clear()           # → forget everything
```

`@cache` (3.9+) is the same with no size limit.

--

## Real-world decorators — retry with backoff

```python
import time, functools

def retry(times=3, backoff=0.5):
    def decorator(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            last = None
            for attempt in range(times):
                try:
                    return fn(*args, **kwargs)
                except Exception as e:
                    last = e
                    time.sleep(backoff * (2 ** attempt))
            raise last
        return wrapper
    return decorator
```

--

## Using `@retry`

```python
@retry(times=5, backoff=0.5)
def fetch(url): ...

fetch("https://flaky.example.com")     # retries up to 5 times,
                                       # sleeping 0.5s, 1s, 2s, 4s, 8s
```

For real production code, reach for [`tenacity`](https://github.com/jd/tenacity) — it adds jitter, exception filtering, retry limits by time, and async support. The decorator above is the educational version.

---

## Class decorators

```python
def add_repr(cls):
    def __repr__(self):
        attrs = ", ".join(f"{k}={v!r}" for k, v in vars(self).items())
        return f"{cls.__name__}({attrs})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

Point(1, 2)                 # Point(x=1, y=2)
```

`@dataclass` is exactly this pattern, but more thorough.

---

## Stacking decorators

```python
@cache
@retry(times=3)
@log_call
def fetch(url): ...
```

Applied bottom-up:

```python
fetch = cache(retry(times=3)(log_call(fetch)))
```

The closest decorator runs first.

---

## Descriptors

Objects that customize attribute access on a class.

```python
class Validated:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return obj.__dict__[self.name]

    def __set__(self, obj, value):
        if value < 0:
            raise ValueError(f"{self.name} must be non-negative")
        obj.__dict__[self.name] = value

class Account:
    balance = Validated()

a = Account()
a.balance = 10              # → a.balance is 10
a.balance                   # 10
a.balance = -1              # ValueError: balance must be non-negative
```

---

## How descriptors work

When you write `obj.attr`, Python checks:

1. The data descriptors on the class (define `__set__` or `__delete__`)
2. `obj.__dict__`
3. Non-data descriptors on the class (only `__get__`)
4. `__getattr__`

`@property` is a descriptor. So is every method (functions are descriptors that bind `self`).

---

## __getattr__

Called when normal attribute lookup **fails**.

```python
class LazyConfig:
    def __getattr__(self, name):
        return os.environ.get(name.upper(), "")

cfg = LazyConfig()
cfg.database_url            # → os.environ['DATABASE_URL']
```

Useful for proxies, lazy loading, dynamic APIs.

---

## __getattribute__

Called for **every** attribute access — including ones that exist.

```python
class Logged:
    def __getattribute__(self, name):
        print(f"accessing {name!r}")
        return super().__getattribute__(name)

obj = Logged()
obj.x = 1                   # → obj.x is 1
obj.x                       # prints "accessing 'x'"; returns 1
```

Easy to break things with this; reach for `__getattr__` first.

---

## Dynamic attribute access

```python
class User:
    name = "Alice"

u = User()
getattr(u, "name")          # 'Alice'
getattr(u, "missing", "?")  # '?'
hasattr(u, "name")          # True
setattr(u, "age", 30)       # → u.age is 30
vars(u)                     # {'age': 30}     (same as u.__dict__)
delattr(u, "age")           # → u.age gone
vars(u)                     # {}
```

These are the building blocks of frameworks like Django.

---

## Classes are objects too

```python
type(42)                    # int
type(int)                   # type
type(type)                  # type
```

`int` is itself an instance — of `type`. **Type is the metaclass**.

---

## Creating classes dynamically

`type(name, bases, namespace)` builds a class on the fly.

```python
Point = type("Point", (), {
    "__init__": lambda self, x, y: setattr(self, "x", x) or setattr(self, "y", y),
    "describe": lambda self: f"({self.x}, {self.y})",
})

Point(1, 2).describe()      # '(1, 2)'
```

Equivalent to `class Point: ...`.

---

## Custom metaclasses

A metaclass is the **class of a class**.

```python
class TableMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        cls.table_name = name.lower()
        return cls

class User(metaclass=TableMeta):
    pass

User.table_name             # 'user'
```

Runs once, at class definition time.

---

## __init_subclass__

The modern alternative to most metaclasses.

```python
class Plugin:
    registry = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry[cls.__name__] = cls

class CSVPlugin(Plugin): pass
class JSONPlugin(Plugin): pass

Plugin.registry
# {'CSVPlugin': <class ...>, 'JSONPlugin': <class ...>}
```

Reach for this **first**. Metaclasses are heavy machinery; `__init_subclass__` covers 90% of use cases.

---

## __set_name__

Lets a descriptor learn its own attribute name.

```python
class Field:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return obj.__dict__[self.name]

class User:
    email = Field()

# Field instance learns it's named 'email' — no manual passing.
```

The mechanism Django and SQLAlchemy use to wire up model fields.

---

## inspect

Read code at runtime.

```python
import inspect

inspect.signature(my_func)
# (a: int, b: int = 0) -> int

inspect.getsource(my_func)
# 'def my_func(a, b=0):\n    return a + b\n'

for name, member in inspect.getmembers(MyClass, inspect.isfunction):
    print(name)
```

Used by `pytest`, `click`, FastAPI to introspect signatures.

---

## eval, exec, compile

```python
eval("2 + 2")               # 4   (evaluates an expression)
exec("x = 5")               # None  (runs statements; here, defines x)
compile("a + b", "<str>", "eval")    # <code object ...>  (re-usable)
```

Powerful and **dangerous**: never run user-supplied strings without sandboxing. Almost always there's a better tool.

---

## Real-world: Django models

```python
class User(models.Model):
    name = models.CharField(max_length=80)
    email = models.EmailField()
```

Django uses a **metaclass** + descriptor `Field` objects to:

- Build a `_meta` registry of fields
- Generate SQL DDL for migrations
- Wire up the manager (`User.objects`)

---

## Real-world: dataclasses

`@dataclass` reads the class's `__annotations__`, generates `__init__`, `__repr__`, `__eq__`, and rewrites the class. All at decoration time, no metaclass needed.

```python
@dataclass
class Point:
    x: float
    y: float
# generated __init__:
#   def __init__(self, x: float, y: float):
#       self.x = x
#       self.y = y
```

---

## Real-world: pytest fixtures

```python
@pytest.fixture
def user():
    return User(name="Alice")

def test_login(user):       # ← pytest inspects signature, injects fixture
    assert user.name == "Alice"
```

`pytest` introspects each test's parameter names to discover fixtures by name. Pure metaprogramming.

---

## Real-world: FastAPI dependencies

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{id}")
def read(id: int, db = Depends(get_db)):
    return db.get(User, id)
```

FastAPI inspects the handler's signature to wire up `Depends`, type-coerce `id`, validate the body, and produce OpenAPI docs.

---

## When NOT to metaprogram

- The code is **harder to read** than the duplication it removes
- A first-class function or a regular class is enough
- Static analysis (mypy, ruff) breaks
- You catch yourself fighting Python's data model

> Metaprogramming is most powerful when it's invisible. If your users have to learn it to use your API, redesign.

---

## What's next

- **Refactoring** — practical code improvement
- **Patterns** — when to reach for them, and when first-class functions suffice
