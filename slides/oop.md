---
title: Object-oriented programming
---

![Class hierarchy diagram](/assets/images/topics/oop.svg)
<!-- .element: class="title-illustration" -->

# OOP in Python

Classes, inheritance, dunders, dataclasses, ABCs.

---

## Defining a class

```python
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def greet(self):
        return f"Hi, I'm {self.name}"

u = User("Alice", "a@example.com")
u.greet()                   # "Hi, I'm Alice"
```

`__init__` is the initializer. `self` is the instance — by convention, always the first parameter.

---

## Class vs instance attributes

```python
class Counter:
    total = 0               # class attribute (shared)

    def __init__(self):
        self.count = 0      # instance attribute (per-instance)

    def increment(self):
        self.count += 1
        Counter.total += 1

c1, c2 = Counter(), Counter()
c1.increment(); c1.increment()
c2.increment()
c1.count, c2.count, Counter.total       # (2, 1, 3)
```

Mutable class attributes are a foot-gun (everyone shares them) — use them only for true constants or counters you mean to share.

---

## Methods: instance, class, static

```python
class Pizza:
    def __init__(self, ingredients):
        self.ingredients = ingredients

    def describe(self):                  # instance
        return ", ".join(self.ingredients)

    @classmethod
    def margherita(cls):                 # class — receives the class
        return cls(["mozzarella", "tomato"])

    @staticmethod
    def is_vegetarian(ingredients):      # plain function in class namespace
        return "pepperoni" not in ingredients

Pizza.margherita().describe()            # 'mozzarella, tomato'
Pizza.is_vegetarian(["mushrooms"])       # True
```

---

## Properties

Make attribute access run code, transparently.

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def fahrenheit(self):
        return self._celsius * 9 / 5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value):
        self._celsius = (value - 32) * 5 / 9

t = Temperature(100)
t.fahrenheit                # 212.0
t.fahrenheit = 32           # → t._celsius is 0
t.fahrenheit                # 32.0
```

---

## Encapsulation conventions

Python has **no real private** attributes — only conventions:

| Prefix | Meaning |
| --- | --- |
| `name` | public |
| `_name` | "internal" — don't touch from outside |
| `__name` | name-mangled to `_ClassName__name` (rarely useful) |

```python
class Account:
    def __init__(self):
        self._balance = 0    # internal — but accessible if you must
```

We are all consenting adults.

---

## Inheritance

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "..."

class Dog(Animal):
    def speak(self):
        return "Woof"

class Puppy(Dog):
    def speak(self):
        return f"Tiny {super().speak().lower()}"

Puppy("Rex").speak()        # "Tiny woof"
```

`super()` calls the next method up the chain.

---

## Multiple inheritance

```python
class Swimmer:
    def move(self): return "swims"

class Flyer:
    def move(self): return "flies"

class Duck(Swimmer, Flyer):
    pass

Duck().move()               # "swims"
```

The first base wins. The full lookup order is the **MRO**.

---

## Method resolution order

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

D.__mro__
# (D, B, C, A, object)
```

Computed by the **C3 linearization** algorithm. `super()` walks this list — never just "the parent".

---

## Mixins

Small classes that add behavior, never instantiated alone.

```python
class JsonMixin:
    def to_json(self):
        import json
        return json.dumps(vars(self))

class TimestampedMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.created_at = datetime.utcnow()

class Article(JsonMixin, TimestampedMixin):
    def __init__(self, title):
        super().__init__()
        self.title = title

a = Article("Hello world")
a.title                     # 'Hello world'
a.created_at                # datetime.datetime(...)
a.to_json()                 # '{"created_at": "...", "title": "Hello world"}'
```

---

## Abstract base classes

Mark a class as not directly instantiable; require subclasses to implement methods.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14159 * self.r ** 2

Shape()       # TypeError: Can't instantiate abstract class
Circle(2)     # OK
```

---

## Protocols (structural typing)

Duck typing made checkable. No inheritance required.

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

def render(thing: Drawable) -> None:
    thing.draw()

class Circle:                   # no `Drawable` base
    def draw(self): print("○")

render(Circle())                # type-check passes
```

A `Protocol` says "anything with these methods qualifies".

---

## Dataclasses

Drop the boilerplate.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x                             # 1.0
p == Point(1.0, 2.0)            # True (auto-generated __eq__)
print(p)                        # Point(x=1.0, y=2.0)
```

You get `__init__`, `__repr__`, `__eq__` for free.

---

## Dataclass options

```python
@dataclass(frozen=True, slots=True, kw_only=True)
class User:
    name: str
    email: str
    age: int = 0

User(name="A", email="a@x", age=30)   # only via keyword args
# instance is immutable, has __slots__
```

- `frozen=True` — makes instances hashable & immutable
- `slots=True` — saves memory, blocks new attributes
- `kw_only=True` — every field is keyword-only

--

## Dataclass `field()`

Use `field()` for non-trivial defaults and per-field options.

```python
from dataclasses import dataclass, field

@dataclass
class Cart:
    user_id: int
    items: list[str] = field(default_factory=list)   # NEW list per instance
    total: float = field(default=0.0, repr=False)    # excluded from __repr__
    _id: str = field(init=False, default_factory=lambda: uuid4().hex)

c = Cart(user_id=1)
c                           # Cart(user_id=1, items=[])   (total hidden)
c.items.append("book")
Cart(user_id=2).items       # []   (independent list, not shared)
```

Never use `items: list[str] = []` — that's the mutable-default-arg trap on a class.

--

## Dataclass `__post_init__`

Hook for derived attributes and validation.

```python
@dataclass
class Range:
    low: int
    high: int
    width: int = field(init=False)

    def __post_init__(self):
        if self.low > self.high:
            raise ValueError("low > high")
        self.width = self.high - self.low

r = Range(1, 10)
r.width                     # 9
Range(10, 1)                # ValueError: low > high
```

---

## __slots__

Skip per-instance `__dict__`; declare allowed attributes.

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
p.z = 3        # AttributeError
```

Smaller memory footprint, faster attribute access. Cannot mix with arbitrary new attributes.

---

## Magic methods (dunders)

Customize how Python's syntax interacts with your objects.

| Dunder | Triggered by |
| --- | --- |
| `__init__`, `__new__` | Construction |
| `__repr__`, `__str__` | `repr(x)`, `str(x)`, `print(x)` |
| `__eq__`, `__hash__` | `==`, `hash()`, dict/set use |
| `__lt__`, `__le__`, `__gt__`, `__ge__` | Ordering |
| `__add__`, `__mul__`, ... | Operators |
| `__len__`, `__iter__`, `__getitem__` | Containers |
| `__enter__`, `__exit__` | `with` blocks |
| `__call__` | `obj()` |

---

## __repr__ vs __str__

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):
        return f"Point({self.x!r}, {self.y!r})"

    def __str__(self):
        return f"({self.x}, {self.y})"

p = Point(1, 2)
print(p)        # (1, 2)        ← __str__
p               # Point(1, 2)   ← __repr__
```

`__repr__` should be unambiguous; `__str__` is for humans.

---

## Equality and hashing

```python
@dataclass(frozen=True)
class Currency:
    code: str
    rate: float

c1 = Currency("USD", 1.0)
c2 = Currency("USD", 1.0)
c1 == c2                # True
{c1, c2}                # {Currency(code='USD', rate=1.0)}
```

Rule: **if you override `__eq__`, override `__hash__` too** (or set it to `None` for unhashable).

---

## Operator overloading

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

Vector(1, 2) + Vector(3, 4)     # Vector(4, 6)
Vector(1, 2) * 3                # Vector(3, 6)
```

---

## Container protocols

Make your class behave like a list or dict.

```python
class Deck:
    def __init__(self):
        self.cards = build_deck()

    def __len__(self):
        return len(self.cards)

    def __getitem__(self, i):
        return self.cards[i]

    def __iter__(self):
        return iter(self.cards)

    def __contains__(self, card):
        return card in self.cards
```

--

## Container protocols — usage

```python
deck = Deck()
len(deck)                    # 52
deck[0]                      # Card(...)
"Ace of Spades" in deck      # True
for card in deck: ...        # iterates all 52
```

Define `__getitem__` and you also get iteration *for free* — Python falls back to indexed iteration if `__iter__` is missing.

---

## Context manager protocol

```python
class Database:
    def __enter__(self):
        self.conn = connect()
        return self.conn

    def __exit__(self, exc_type, exc, tb):
        self.conn.close()
        return False         # don't swallow exceptions

with Database() as db:
    db.query(...)
```

Or use `@contextlib.contextmanager` on a generator function.

---

## Callable objects

```python
class Adder:
    def __init__(self, n):
        self.n = n

    def __call__(self, x):
        return x + self.n

add5 = Adder(5)
add5(10)                # 15
callable(add5)          # True
```

A class that defines `__call__` is itself callable.

---

## Enums

```python
from enum import Enum, auto

class Status(Enum):
    PENDING = auto()
    ACTIVE = auto()
    CLOSED = auto()

Status.ACTIVE               # <Status.ACTIVE: 2>
Status.ACTIVE.name          # 'ACTIVE'
Status.ACTIVE.value         # 2
list(Status)
# → [<Status.PENDING: 1>, <Status.ACTIVE: 2>, <Status.CLOSED: 3>]
```

--

## Enums — string values

For enums that compare to plain strings:

```python
class Role(str, Enum):
    ADMIN = "admin"
    USER  = "user"

Role.ADMIN                  # <Role.ADMIN: 'admin'>
Role.ADMIN.value            # 'admin'
Role.ADMIN == "admin"       # True   (str-Enum compares to plain strings)
```

Mixing in `str` makes the enum value JSON-serializable and DB-friendly without `.value` everywhere.

---

## Composition over inheritance

Prefer holding an object to extending it.

```python
class Engine:
    def start(self): ...

class Car:
    def __init__(self):
        self.engine = Engine()           # composition

    def start(self):
        self.engine.start()              # delegation
```

vs.

```python
class Car(Engine):                       # rarely the right model
    pass
```

A `Car` *has* an engine; it isn't *a kind of* engine.

---

## SOLID, briefly

- **S**ingle Responsibility — one reason to change
- **O**pen/Closed — open to extension, closed to modification
- **L**iskov Substitution — subtypes must honor the parent's contract
- **I**nterface Segregation — small, focused Protocols
- **D**ependency Inversion — depend on abstractions, inject concretes

We'll revisit these in **Refactoring**.

---

## What's next

- **Metaprogramming** — how decorators, ORMs, and `pytest` actually work
- **Refactoring** — practical code improvement
