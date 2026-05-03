---
title: Design patterns
---

![Repeating pattern of blocks](/assets/images/topics/patterns.svg)
<!-- .element: class="title-illustration" -->

# Design patterns

In Python, half of them disappear.

---

## Why patterns?

A shared vocabulary for recurring designs:

- "Use a Strategy here" is shorter than three paragraphs
- They name solutions, not problems
- They're a starting point — not a target

---

## Why "half disappear"?

Python has:

- First-class functions
- Dynamic typing
- Duck typing / Protocols
- Closures
- Decorators
- Context managers

Many GoF patterns work around limitations Python doesn't have.

---

## Categories

| Group | Solves... |
| --- | --- |
| **Creational** | How objects are made |
| **Structural** | How objects are composed |
| **Behavioral** | How objects communicate |

Press **↓** for which patterns belong to each group.

--

## Patterns by group

| Creational | Structural | Behavioral |
| --- | --- | --- |
| Factory method | Adapter | Strategy |
| Abstract Factory | Decorator | Observer |
| Builder | Facade | Command |
| Singleton | Proxy | State |
| Prototype | Composite | Iterator |
|  |  | Template method |
|  |  | Chain of responsibility |

--

## Why the buckets matter

Interview questions like "name a behavioral pattern" or "what's the difference between Decorator and Proxy?" become much easier when you recognize the group first.

- "Behavioral" → narrows to ~7 candidates instead of 17
- "Decorator vs Proxy" → both are Structural; the question is *what they wrap* and *why*

---

## Strategy

![Strategy pattern](/assets/images/patterns/strategy.svg)
<!-- .element: class="pattern-illustration" -->

Plug-replaceable algorithm.

--

## Strategy — class-based (Java/C# style)

```python
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data): ...

class QuickSort(SortStrategy): ...
class MergeSort(SortStrategy): ...
```

Interface plus a class per algorithm. Verbose, but the only option in statically-typed languages without first-class functions.

--

## Strategy — Pythonic

```python
# A function IS the strategy
def quicksort(data): ...
def mergesort(data): ...

def run(data, strategy):
    return strategy(data)

run(items, quicksort)         # use quicksort
run(items, mergesort)         # swap implementation
```

No interface, no class hierarchy — Python's first-class functions are the abstraction.

---

## Factory method

![Factory method pattern](/assets/images/patterns/factory-method.svg)
<!-- .element: class="pattern-illustration" -->

Hide the construction logic behind a callable.

--

## Factory method — code

```python
class Pizza:
    def __init__(self, ingredients): ...

    @classmethod
    def margherita(cls):
        return cls(["mozzarella", "tomato"])

    @classmethod
    def from_dict(cls, data):
        return cls(ingredients=data["ingredients"])

p = Pizza.margherita()                # Pizza(['mozzarella', 'tomato'])
q = Pizza.from_dict({"ingredients": ["pepperoni"]})
```

`@classmethod` factories are *the* Python idiom.

---

## Abstract Factory

![Abstract Factory pattern](/assets/images/patterns/abstract-factory.svg)
<!-- .element: class="pattern-illustration" -->

A family of related factories — switch the family, swap all products.

--

## Abstract Factory — code

```python
# postgres_repos.py
def make_user_repo(): return PostgresUserRepo()
def make_order_repo(): return PostgresOrderRepo()

# memory_repos.py
def make_user_repo(): return InMemoryUserRepo()
def make_order_repo(): return InMemoryOrderRepo()

# Switch implementations by importing:
from postgres_repos import make_user_repo
```

Or pass a registry/dict — Python doesn't need a class hierarchy for this.

---

## Builder

![Builder pattern](/assets/images/patterns/builder.svg)
<!-- .element: class="pattern-illustration" -->

Step-by-step construction of complex objects. In Python, kwargs and dataclasses usually suffice.

--

## Builder — code

```python
@dataclass
class Pizza:
    size: str = "medium"
    cheese: str = "mozzarella"
    toppings: list[str] = field(default_factory=list)
    sauce: str = "tomato"

p = Pizza(
    size="large",
    toppings=["mushrooms", "olives"],
    sauce="white",
)
```

Reach for a real Builder only when validation is staged across several setup steps.

---

## Singleton

![Singleton pattern](/assets/images/patterns/singleton.svg)
<!-- .element: class="pattern-illustration" -->

One instance, shared across all callers.

--

## Singleton — class form

```python
class Config:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

Config() is Config()        # True   (always the same instance)
```

You almost never want this form in Python.

--

## Singleton — Pythonic (a module)

```python
# config.py
DATABASE_URL = "postgres://..."
DEBUG = False
```

```python
# elsewhere
import config
print(config.DATABASE_URL)             # postgres://...
```

Modules are imported once and cached in `sys.modules`. Subsequent `import` statements return the same object — that's a singleton, with no `__new__` gymnastics.

---

## Prototype

![Prototype pattern](/assets/images/patterns/prototype.svg)
<!-- .element: class="pattern-illustration" -->

"Clone an existing instance." Built into the standard library.

--

## Prototype — code

```python
import copy

c1 = Config(environment="prod")
c2 = copy.deepcopy(c1)                 # independent copy
c2.environment = "staging"             # → c2 changes, c1 unaffected
c1.environment, c2.environment         # ('prod', 'staging')
```

That's the pattern. No abstract `Prototype` class needed.

---

## Adapter

![Adapter pattern](/assets/images/patterns/adapter.svg)
<!-- .element: class="pattern-illustration" -->

Make two incompatible interfaces work together.

--

## Adapter — wrapper class

```python
# A class with .write() — but we need a .send() interface
class FileWriter:
    def write(self, data): ...

class FileSender:
    def __init__(self, fw): self.fw = fw
    def send(self, data):
        self.fw.write(data)
```

--

## Adapter — the one-liner

If all you need is a method renamed, skip the wrapper class:

```python
from types import SimpleNamespace

fw = FileWriter()
sender = SimpleNamespace(send=fw.write)
sender.send(b"...")           # → calls fw.write(b"...")
```

`__getattr__` delegation is the next step up if you need to forward many methods.

---

## Decorator (structural)

![Decorator pattern](/assets/images/patterns/decorator.svg)
<!-- .element: class="pattern-illustration" -->

Wrap an object to add behavior. Conceptually identical to function decorators, just on instances.

--

## Decorator — code

```python
class Coffee:
    def cost(self): return 5
    def description(self): return "Coffee"

class Milk:
    def __init__(self, drink):
        self.drink = drink
    def cost(self):
        return self.drink.cost() + 1
    def description(self):
        return f"{self.drink.description()}, milk"

drink = Milk(Coffee())
drink.cost(), drink.description()    # (6, 'Coffee, milk')
```

---

## Facade

![Facade pattern](/assets/images/patterns/facade.svg)
<!-- .element: class="pattern-illustration" -->

A simple front for a complex subsystem.

--

## Facade — code

```python
# subsystems
def boot_disk(): ...
def boot_cpu(): ...
def boot_memory(): ...
def boot_io(): ...

# facade
def boot_computer():
    boot_disk(); boot_cpu(); boot_memory(); boot_io()
```

Callers use `boot_computer()`; they don't need to know the order.

---

## Proxy

![Proxy pattern](/assets/images/patterns/proxy.svg)
<!-- .element: class="pattern-illustration" -->

Stand-in object that controls access — lazy load, logging, or remote.

--

## Proxy — code

```python
class LazyImage:
    def __init__(self, path):
        self.path = path
        self._img = None

    def show(self):
        if self._img is None:
            self._img = load_image(self.path)
        self._img.display()
```

`__getattr__` is the cleanest way to forward unknown method calls to the wrapped object.

---

## Composite

![Composite pattern](/assets/images/patterns/composite.svg)
<!-- .element: class="pattern-illustration" -->

Treat a tree of objects uniformly.

--

## Composite — code

```python
class File:
    def __init__(self, name, size): self.name, self.size = name, size
    def total_size(self): return self.size

class Directory:
    def __init__(self, name): self.name = name; self.children = []
    def total_size(self): return sum(c.total_size() for c in self.children)
```

Both expose `.total_size()`. The caller doesn't care which it has.

---

## Observer

![Observer pattern](/assets/images/patterns/observer.svg)
<!-- .element: class="pattern-illustration" -->

Object A notifies subscribers when its state changes.

--

## Observer — code

```python
class EventBus:
    def __init__(self): self.handlers = defaultdict(list)
    def on(self, event, fn): self.handlers[event].append(fn)
    def emit(self, event, *args, **kwargs):
        for fn in self.handlers[event]:
            fn(*args, **kwargs)

bus = EventBus()
bus.on("user.created", send_welcome_email)
bus.on("user.created", create_default_workspace)
bus.emit("user.created", user)         # both handlers run, in order
```

Django ships this as **signals**.

---

## Command

![Command pattern](/assets/images/patterns/command.svg)
<!-- .element: class="pattern-illustration" -->

Encapsulate a request as an object — easy to queue, log, or undo.

--

## Command — code

```python
@dataclass
class TransferMoney:
    src: Account
    dst: Account
    amount: Decimal

    def execute(self):
        self.src.withdraw(self.amount)
        self.dst.deposit(self.amount)

cmd = TransferMoney(a, b, Decimal("100"))
audit_log.append(cmd)                  # → cmd recorded
queue.put(cmd)                         # → cmd scheduled
cmd.execute()                          # → withdraws then deposits
```

The cornerstone of CQRS, undo systems, and task queues.

---

## State

![State pattern](/assets/images/patterns/state.svg)
<!-- .element: class="pattern-illustration" -->

Behavior changes with state. Replace flag-soup with state objects.

--

## State — code

```python
class Draft:
    def publish(self, doc): doc.state = Published()

class Published:
    def publish(self, doc): raise InvalidTransition()
    def archive(self, doc): doc.state = Archived()

class Archived:
    def archive(self, doc): raise InvalidTransition()

class Document:
    def __init__(self): self.state = Draft()
    def publish(self): self.state.publish(self)
```

Each transition is enforced by the state, not by `if` chains.

---

## Iterator

![Iterator pattern](/assets/images/patterns/iterator.svg)
<!-- .element: class="pattern-illustration" -->

You already use it. Every `for` loop is an iterator.

--

## Iterator — code

```python
class Range:
    def __init__(self, n): self.n = n
    def __iter__(self):
        i = 0
        while i < self.n:
            yield i
            i += 1

list(Range(4))               # [0, 1, 2, 3]
```

Generators (`yield`) are the standard way; the explicit class form is rarely needed.

---

## Template method

![Template method pattern](/assets/images/patterns/template-method.svg)
<!-- .element: class="pattern-illustration" -->

Skeleton in the base class, hooks in subclasses.

--

## Template method — code

```python
class ReportPipeline:
    def run(self):
        data = self.fetch()
        cleaned = self.clean(data)
        self.publish(cleaned)

    def fetch(self): raise NotImplementedError
    def clean(self, data): return data
    def publish(self, data): raise NotImplementedError
```

Subclasses override `fetch`, `clean`, `publish`. The flow lives in `run`.

---

## Chain of responsibility

![Chain of responsibility pattern](/assets/images/patterns/chain-of-responsibility.svg)
<!-- .element: class="pattern-illustration" -->

A chain of handlers; each decides whether to handle or pass on.

--

## Chain of responsibility — handlers

```python
def auth(req):
    if not req.user:
        return error("403")
    return None                  # pass on

def rate_limit(req):
    if exceeded(req):
        return error("429")
    return None

def serve(req):
    return ok(req)               # final handler
```

--

## Chain of responsibility — running the chain

```python
def run(req, *handlers):
    for h in handlers:
        result = h(req)
        if result is not None:
            return result
    return None

run(req, auth, rate_limit, serve)      # 200 OK, or 403/429 on the way
```

HTTP middleware is exactly this — Django, Flask, FastAPI, and ASGI all implement variants of this pattern.

---

## Pythonic alternatives — at a glance

| GoF pattern | Pythonic form |
| --- | --- |
| Strategy | A function |
| Singleton | A module |
| Prototype | `copy.deepcopy` |
| Iterator | Generators / `__iter__` |
| Command | A `@dataclass` with `.execute()` |
| Decorator | `@decorator` syntax |
| Adapter | A wrapper class or `__getattr__` |
| Observer | Callbacks, signals, events |
| Template Method | Plain inheritance with hook methods |

---

## When patterns help

- The team **shares the vocabulary**
- The shape recurs ≥ 3 times in the codebase
- The pattern is **needed**, not aspirational

> Don't go pattern-hunting. Solve the problem in front of you, then notice if a pattern fell out.

---

## What's next

- **Tooling** — packaging, task runners, code analysis
- **Testing** — `pytest`, fixtures, mocking
