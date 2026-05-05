---
title: Python basics
---

![REPL with a snake curl](/assets/images/topics/python-basics.svg)
<!-- .element: class="title-illustration" -->

# Python basics

The language, in one deck.

---

## Before we write any code — install Python

The recommended toolchain for new projects in 2026 is **`uv`**. One binary that installs Python, scaffolds projects, manages dependencies, and runs commands.

We'll set up everything in four short steps.

Press **↓** to walk through them.

--

## Step 1 — install `uv`

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# macOS (Homebrew)
brew install uv

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Verify:

```bash
uv --version                    # uv 0.5.x
```

`uv` is a single static binary — no Python required to install it.

--

## Step 2 — install Python via `uv`

No need for `pyenv` or `asdf` — `uv` installs CPython itself.

```bash
# Install the latest stable Python
uv python install

# Or pick a specific minor / patch
uv python install 3.13
uv python install 3.12.7

# Several at once
uv python install 3.11 3.12 3.13
```

Verify with **`uv run`** — that's the canonical "is my install working" check:

```bash
uv run python --version             # Python 3.13.1   (uv's latest installed)
```

Several Pythons can live side by side — uv picks the right one per project (we'll pin one in Step 3).

--

## Step 2 — what about `python3 --version`?

`uv python install` puts the new Python in **uv's own storage** (`~/.local/share/uv/python/...`). It does **not** touch your shell's `PATH`.

So this can happen:

```bash
uv python install 3.13              # → installs 3.13.1 into uv storage
python3 --version                   # Python 3.13.7   ← system Python, not uv's!
which python3                       # /usr/local/bin/python3
```

Your shell still resolves `python3` to whatever the OS / Homebrew / python.org installer put there.

--

## Step 2 — that's expected

It's fine. You'll work **inside projects** and use `uv run` — which uses the project's pinned Python automatically:

```bash
uv run python --version             # ← uv's Python, the one your project uses
```

If you really want uv's Pythons on `PATH` directly (so `python3.13` works in any shell), opt in:

```bash
uv python update-shell              # → appends to ~/.zshrc or ~/.bashrc
```

Most workflows don't need this. Stick with `uv run` and you'll never go wrong.

--

## Step 2 — listing what's available

`uv python list` shows two kinds of entries: **installed** (with a path) and **available for download** (marked `<download>`).

```bash
# One entry per minor version (default)
uv python list

# Every patch release available for download
uv python list --all-versions

# Pre-releases too (e.g., 3.14a1, free-threaded builds)
uv python list --all-versions --preview

# Only what's installed (uv-managed AND system-discovered)
uv python list --only-installed
```

Caveat: `--only-installed` includes Pythons uv discovered on your system (Homebrew, python.org installer, `/usr/bin/python3`, etc.) — not just ones uv installed itself. Paths under `~/.local/share/uv/` are uv-managed; everything else is from your OS.

--

## Step 2 — keeping the list fresh

`uv` ships its catalogue of downloadable Pythons inside the binary itself. To get newer releases (e.g., when CPython 3.14 GA's), update `uv`:

```bash
uv self update                  # → new uv → new Python catalogue
uv --version                    # check
uv python list --all-versions   # confirm new entries appear
```

No separate "refresh index" command — `uv self update` is all you need.

--

## Step 3 — scaffold a project

```bash
uv init hello-python            # scaffolds the project
cd hello-python
uv python pin 3.13              # pin to 3.13 (writes .python-version)
```

`uv init` creates:

```
hello-python/
├── pyproject.toml              # project metadata + deps
├── README.md
├── .python-version             # pinned interpreter
└── main.py                     # a "Hello from hello-python!" script
```

`.venv/` is **not** there yet — uv creates it on demand when you run something.

--

## Step 4 — add dependencies and run

```bash
uv run python main.py           # → Hello from hello-python!

uv add requests                  # add a runtime dep
uv add --dev pytest ruff mypy    # add dev-only deps
```

`uv run` ensures the venv is up to date, then runs the command in it. You never manually `source .venv/bin/activate`.

```bash
uv run python -c "import requests; print(requests.__version__)"
# 2.32.3
uv run pytest                    # run tests
uv run ruff check . --fix        # lint + autofix
```

That's the whole workflow. Before we structure things into files, let's play with Python interactively.

---

## The interactive interpreter (REPL)

Python ships with an **interactive shell** — the REPL (Read-Eval-Print-Loop). Type code, see results immediately. Like Ruby's `irb`, but built into Python itself.

```bash
$ uv run python
Python 3.13.1 (main, ...) on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> 2 + 3
5
>>> name = "Alice"
>>> f"Hello, {name}"
'Hello, Alice'
>>> exit()                       # or press Ctrl-D
```

Use it for quick experiments, exploring a library, debugging a snippet, sanity-checking syntax.

--

## Why `uv run python`?

Plain `python3` uses whatever's on your `PATH` — usually the system Python, with no access to your project's installed packages.

`uv run python` uses the project's pinned Python **and** its `.venv/`:

```bash
uv add requests                  # add a dep to the project
uv run python                    # REPL with the project's env
>>> import requests
>>> r = requests.get("https://httpbin.org/get")
>>> r.json()["url"]
'https://httpbin.org/get'
```

Inside a uv project, **always** use `uv run python` for the REPL.

--

## A better REPL — IPython

IPython adds tab completion, syntax highlighting, history, multi-line editing, and "magic" commands.

```bash
uv add --dev ipython
uv run ipython
```

```ipython
In [1]: ?str.split               # quick docs (note the leading ?)
In [2]: %timeit sum(range(1_000_000))
12.6 ms ± 89.7 µs per loop (mean ± std. dev. of 7 runs, ...)
In [3]: %paste                   # paste indented code without breaking
In [4]: history                  # previous commands this session
```

Once you've used it, the plain REPL feels stark. Many editors (VS Code, PyCharm, Jupyter) embed IPython internally.

--

## Drop into the REPL after a script

```bash
uv run python -i main.py
```

`-i` runs the script, then **keeps the interpreter open** with everything the script defined still in scope. Useful for poking at state after a function ran.

```python
# main.py
import requests
data = requests.get("https://httpbin.org/get").json()
```

```bash
$ uv run python -i main.py
>>> data["url"]                  # `data` is still here — inspect it
'https://httpbin.org/get'
>>> requests.__version__
'2.32.3'
```

---

## Variables

Python is **dynamically typed**. No declaration needed — assignment creates the binding.

```python
x = 42
name = "Alice"
items = [1, 2, 3]
```

Names are conventionally `snake_case`. Constants are `UPPER_SNAKE_CASE` (by convention only — Python has no real `const`).

---

## Built-in types

| Type | Examples |
| --- | --- |
| `int` | `42`, `0`, `-7`, `1_000_000` |
| `float` | `3.14`, `2.0`, `1e-5` |
| `bool` | `True`, `False` |
| `str` | `"hello"`, `'hello'` |
| `bytes` | `b"hello"` |
| `list` | `[1, 2, 3]` |
| `tuple` | `(1, 2, 3)` |
| `dict` | `{"k": "v"}` |
| `set` | `{1, 2, 3}` |
| `NoneType` | `None` |

---

## Numbers

```python
2 + 3        # 5
10 / 3       # 3.3333333333333335   (true division → float)
10 // 3      # 3                    (floor division)
10 % 3       # 1                    (modulo)
2 ** 10      # 1024                 (exponent)
```

Integers are arbitrary precision:

```python
2 ** 100     # 1267650600228229401496703205376
```

---

## Strings

Strings are **immutable** sequences of Unicode code points.

```python
greeting = "Hello"
greeting + ", world!"        # 'Hello, world!'
greeting * 3                 # 'HelloHelloHello'
"Hi"[0]                      # 'H'
len(greeting)                # 5
```

Single, double, and triple quotes all valid:

```python
s = """multi-line
string"""
```

--

## Strings — common methods

```python
s = "  Hello, World!  "
s.strip()                   # 'Hello, World!'
s.lower()                   # '  hello, world!  '
s.upper()                   # '  HELLO, WORLD!  '
s.replace("Hello", "Hi")    # '  Hi, World!  '

"a,b,c".split(",")          # ['a', 'b', 'c']
",".join(["a", "b", "c"])   # 'a,b,c'

"abc".startswith("ab")      # True
"abc".endswith("c")         # True
"abc".find("b")             # 1   (index of first match, -1 if missing)
"abc".count("a")            # 1
```

--

## Strings — checks and padding

```python
"42".isdigit()              # True
"abc".isalpha()             # True
"abc123".isalnum()          # True
"   ".isspace()             # True

"hi".center(10, "-")        # '----hi----'
"hi".ljust(6, ".")          # 'hi....'
"42".zfill(5)               # '00042'

"snake_case".replace("_", "-")    # 'snake-case'
"path/to/file".rsplit("/", 1)     # ['path/to', 'file']
```

--

## f-strings

The modern way to format strings.

```python
name = "Alice"
age = 30
print(f"{name} is {age} years old")     # Alice is 30 years old

# Expressions, formatting, padding, conversion specifiers:
print(f"{3.14159:.2f}")                  # 3.14
print(f"{42:>5}")                        # '   42'
print(f"{name!r}")                       # 'Alice'    (with quotes)
```

--

## f-strings — format spec mini-language

```python
n = 1234567

# Numbers
f"{n:,}"                    # '1,234,567'    (thousands sep)
f"{n:_}"                    # '1_234_567'
f"{0.5:.0%}"                # '50%'           (percent)
f"{255:08b}"                # '11111111'      (binary, 8-wide, zero-pad)
f"{255:#x}"                 # '0xff'          (hex with prefix)
f"{1234.5:e}"               # '1.234500e+03'  (scientific)

# Alignment & padding
f"{'hi':<10}|"              # 'hi        |'   (left)
f"{'hi':>10}|"              # '        hi|'   (right)
f"{'hi':^10}|"              # '    hi    |'   (center)
f"{42:0>5}"                 # '00042'         (custom pad char)

# Debug syntax (Python 3.8+)
x = 42
f"{x=}"                     # 'x=42'
```

---

## bytes vs str

- `str` — text (Unicode)
- `bytes` — raw octets

```python
"héllo".encode("utf-8")     # b'h\xc3\xa9llo'
b'h\xc3\xa9llo'.decode("utf-8")  # 'héllo'
```

Network and file I/O at the lowest level always speaks `bytes`.

---

## Booleans & truthiness

```python
True and False              # False
True or False               # True
not True                    # False
```

Truthy by default, **falsy values** are: `False`, `None`, `0`, `0.0`, `""`, `[]`, `()`, `{}`, `set()`.

```python
if items:                   # idiomatic "is non-empty"
    process(items)
```

---

## None

`None` is the unique null value.

```python
x = None
x is None                   # True
x == None                   # works, but use `is` for identity
```

Functions that don't `return` anything return `None` implicitly.

---

## Lists

Mutable, ordered, can hold any types.

```python
xs = [1, 2, 3]
xs.append(4)                # → xs is [1, 2, 3, 4]
xs.insert(0, 0)             # → xs is [0, 1, 2, 3, 4]
xs.pop()                    # 4   (xs is [0, 1, 2, 3])
xs[1] = 99                  # → xs is [0, 99, 2, 3]
xs[1:3]                     # [99, 2]    (slice)
xs[::-1]                    # [3, 2, 99, 0]   (reversed copy)
```

Press **↓** for more list operations.

--

## Lists — sorting and searching

```python
xs = [3, 1, 4, 1, 5, 9, 2, 6]
xs.sort()                   # → xs is [1, 1, 2, 3, 4, 5, 6, 9]
xs.reverse()                # → xs is [9, 6, 5, 4, 3, 2, 1, 1]
xs.count(1)                 # 2
xs.index(4)                 # 3   (position of first 4)

# Non-mutating versions return a new list:
sorted([3, 1, 2], reverse=True)     # [3, 2, 1]
list(reversed([1, 2, 3]))           # [3, 2, 1]
```

--

## Lists — combining and copying

```python
xs = [1, 2]
xs.extend([3, 4])           # → xs is [1, 2, 3, 4]
xs + [5, 6]                 # [1, 2, 3, 4, 5, 6]   (new list)
xs * 2                      # [1, 2, 3, 4, 1, 2, 3, 4]

xs.copy()                   # [1, 2, 3, 4]   (shallow copy)
xs[:]                       # [1, 2, 3, 4]   (also a shallow copy)

# Beware shallow copy with nested lists:
import copy
nested = [[1], [2]]
copy.deepcopy(nested)       # [[1], [2]]   (independent inner lists)
```

---

## Tuples

Immutable, often used for fixed-shape records.

```python
point = (3, 4)
x, y = point                # unpacking
point[0]                    # 3
point[0] = 99               # TypeError: tuples are immutable
```

Single-element tuple needs a trailing comma:

```python
(42,)                       # tuple
(42)                        # just an int!
```

--

## Tuples — namedtuple

For tuples with named fields:

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x                         # 3
p[0]                        # 3   (still indexable)
p._asdict()                 # {'x': 3, 'y': 4}
p._replace(x=9)             # Point(x=9, y=4)   (new instance)
```

For more features (defaults, type hints), reach for `typing.NamedTuple` or `@dataclass(frozen=True)`.

--

## Tuples — unpacking patterns

```python
# Star-unpacking captures the rest
first, *rest = [1, 2, 3, 4]
first, rest                 # (1, [2, 3, 4])

*head, last = [1, 2, 3, 4]
head, last                  # ([1, 2, 3], 4)

# Swapping
a, b = 1, 2
a, b = b, a                 # → a is 2, b is 1

# Nested unpacking
(x, y), z = (1, 2), 3       # x=1, y=2, z=3
```

---

## Sets

Unordered collections of unique hashable elements.

```python
s = {1, 2, 3}
s.add(4)                    # → s is {1, 2, 3, 4}
s.add(2)                    # → s is {1, 2, 3, 4} (already in, no change)
2 in s                      # True

a = {1, 2, 3}
b = {3, 4, 5}
a | b                       # {1, 2, 3, 4, 5}  union
a & b                       # {3}              intersection
a - b                       # {1, 2}           difference
```

---

## Dicts

Hash maps. Keys must be hashable; values can be anything.

```python
user = {"name": "Alice", "age": 30}
user["name"]                # 'Alice'
user["email"] = "a@x"       # → user gains key 'email'
user.get("missing", "?")    # '?'  (default returned)
"name" in user              # True
del user["age"]             # → user loses key 'age'
list(user.items())          # [('name', 'Alice'), ('email', 'a@x')]
```

Insertion order is preserved (since Python 3.7).

--

## Dicts — merging

```python
a = {"x": 1, "y": 2}
b = {"y": 99, "z": 3}

# Modern (3.9+)
a | b                       # {'x': 1, 'y': 99, 'z': 3}   (new dict)
a |= b                      # → a is {'x': 1, 'y': 99, 'z': 3}

# Spread operator (any version)
{**a, **b}                  # {'x': 1, 'y': 99, 'z': 3}

# Update in place
a.update(b)                 # → a is {'x': 1, 'y': 99, 'z': 3}
```

Right-hand wins on key collisions.

--

## Dicts — defaultdict and Counter

```python
from collections import defaultdict, Counter

# defaultdict — auto-creates missing keys
groups = defaultdict(list)
groups["even"].append(2)
groups["even"].append(4)
groups["odd"].append(1)
dict(groups)                # {'even': [2, 4], 'odd': [1]}

# Counter — frequency map
Counter("mississippi")
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
Counter("mississippi").most_common(2)
# [('i', 4), ('s', 4)]
```

---

## Comprehensions

Concise transformations of iterables.

```python
squares = [x * x for x in range(5)]
# squares == [0, 1, 4, 9, 16]

evens = [x for x in range(10) if x % 2 == 0]
# evens == [0, 2, 4, 6, 8]

squared = {x: x * x for x in range(5)}
# squared == {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

text = "Hello hello WORLD"
unique_words = {w.lower() for w in text.split()}
# unique_words == {'hello', 'world'}
```

---

## Generator expressions

Like a list comprehension, but lazy.

```python
total = sum(x * x for x in range(1_000_000))
# total == 333332833333500000
# (doesn't build a list in memory — values are produced on demand)
```

Use parentheses, no brackets.

---

## Control flow

```python
score = 85
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
else:
    grade = "C"
grade                                    # 'B'

# Ternary
n = 7
parity = "even" if n % 2 == 0 else "odd" # 'odd'
```

---

## Loops

```python
for x in [1, 2, 3]:
    print(x)                     # 1, then 2, then 3

for i in range(5):
    print(i)                     # 0, 1, 2, 3, 4

names = ["Alice", "Bob"]
for i, name in enumerate(names):
    print(i, name)               # 0 Alice, then 1 Bob

xs, ys = [1, 2, 3], ["a", "b", "c"]
for a, b in zip(xs, ys):
    print(a, b)                  # 1 a, then 2 b, then 3 c

while not done:                  # loops until `done` becomes truthy
    step()
```

`break`, `continue`, and a `for ... else` clause that runs if the loop wasn't broken out of.

---

## match statement

Structural pattern matching (Python 3.10+).

```python
def describe(point):
    match point:
        case (0, 0):
            return "origin"
        case (x, 0):
            return f"on x-axis at {x}"
        case (0, y):
            return f"on y-axis at {y}"
        case (x, y) if x == y:
            return "diagonal"
        case _:
            return "somewhere else"

describe((0, 0))            # 'origin'
describe((5, 0))            # 'on x-axis at 5'
describe((3, 3))            # 'diagonal'
describe((1, 7))            # 'somewhere else'
```

---

## Functions

```python
def add(a, b):
    return a + b

def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

greet("Alice")              # 'Hello, Alice!'
greet("Bob", "Hi")          # 'Hi, Bob!'
greet("Bob", greeting="Hi") # keyword argument
```

---

## *args and **kwargs

Capture variable positional and keyword arguments.

```python
def log(*args, **kwargs):
    print("positional:", args)
    print("keyword:", kwargs)

log(1, 2, 3, level="INFO")
# positional: (1, 2, 3)
# keyword: {'level': 'INFO'}
```

Useful for forwarding arguments to wrapped callables.

---

## Positional-only and keyword-only

```python
def f(pos_only, /, normal, *, kw_only):
    ...

f(1, 2, kw_only=3)          # OK
f(pos_only=1, ...)          # TypeError: pos_only is positional-only
f(1, 2, 3)                  # TypeError: kw_only must be keyword
```

`/` ends positional-only; `*` starts keyword-only.

--

## The mutable-default-arg trap

A classic Python gotcha:

```python
def append_to(item, target=[]):       # ← list created ONCE,
    target.append(item)                # at function-definition time
    return target

append_to(1)                # [1]
append_to(2)                # [1, 2]   ← surprise!
append_to(3)                # [1, 2, 3]
```

Default arguments are evaluated **once**, when `def` runs. Mutating them leaks state across calls.

--

## Mutable defaults — the fix

Use `None` as a sentinel and create the mutable inside the body:

```python
def append_to(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target

append_to(1)                # [1]
append_to(2)                # [2]   ← independent per call
```

This rule applies to **any mutable default**: `list`, `dict`, `set`, custom objects.

---

## Type hints

Optional but encouraged. Checked by tools (`mypy`, `pyright`), not by Python itself.

```python
def add(a: int, b: int) -> int:
    return a + b

names: list[str] = []
config: dict[str, int] = {}

from typing import Optional
def find(id: int) -> Optional[User]:
    ...
```

Modern syntax: `int | None` instead of `Optional[int]` (3.10+).

---

## Lambdas

Anonymous one-expression functions.

```python
square = lambda x: x * x
square(5)                   # 25

sorted(users, key=lambda u: u.age)        # users, ordered by age
list(filter(lambda x: x > 0, nums))       # only positive numbers
```

For anything more than one expression, use `def`.

---

## Closures

Inner functions capture variables from their enclosing scope.

```python
def make_adder(n):
    def add(x):
        return x + n
    return add

add5 = make_adder(5)
add5(10)                    # 15
```

Use `nonlocal` to **rebind** an outer variable, not just read it.

---

## Generators

Functions with `yield` produce values lazily.

```python
def count_up_to(n):
    i = 0
    while i < n:
        yield i
        i += 1

for x in count_up_to(3):
    print(x)                # 0, 1, 2
```

Each `yield` pauses the function; the next iteration resumes.

--

## Generators — `yield from`

Delegate iteration to another iterable.

```python
def chain(*iterables):
    for it in iterables:
        yield from it       # equivalent to: for x in it: yield x

list(chain([1, 2], (3, 4), "ab"))    # [1, 2, 3, 4, 'a', 'b']
```

`yield from` also forwards `send()`, `throw()`, and the return value — the right primitive for composing generators.

--

## Generators — `send()`

Two-way communication: callers can push values **into** a paused generator.

```python
def echo():
    while True:
        received = yield
        print(f"got: {received}")

g = echo()
next(g)                     # prime: advance to first yield
g.send("hi")                # prints "got: hi"
g.send("there")             # prints "got: there"
g.close()                   # stop the generator
```

The basis of asyncio's coroutine machinery before `async`/`await` existed.

---

## Iterators

The protocol behind `for` loops.

```python
it = iter([1, 2, 3])
next(it)                    # 1
next(it)                    # 2

class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self): return self
    def __next__(self):
        if self.n <= 0: raise StopIteration
        self.n -= 1
        return self.n + 1
```

---

## Exceptions

```python
try:
    result = risky()
except ValueError as e:
    print(f"bad value: {e}")
except (KeyError, IndexError):
    print("not found")
else:
    print("no exception")
finally:
    cleanup()
```

`raise` to throw; `raise X from Y` to chain.

--

## Custom exceptions

Subclass `Exception` (not `BaseException`).

```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        super().__init__(f"need {amount}, have {balance}")
        self.balance = balance
        self.amount = amount

try:
    raise InsufficientFundsError(balance=10, amount=50)
except InsufficientFundsError as e:
    e.balance, e.amount     # (10, 50)
```

--

## Exception hierarchy

![Python exception hierarchy](/assets/diagrams/exception-hierarchy.dot.svg)

`BaseException` is the root. **Always inherit from `Exception`**, not `BaseException`, so users can catch broadly without swallowing `KeyboardInterrupt`.

--

## Exception chaining

`raise X from Y` records the cause:

```python
try:
    config = json.loads(raw)
except json.JSONDecodeError as e:
    raise ConfigError("invalid config") from e
```

The traceback shows: "*The above exception was the direct cause of the following exception*". Without `from`, you'd get an implicit "*During handling of …*" chain.

Use `from None` to **suppress** the chain:

```python
raise ValueError("bad input") from None
```

--

## Exception groups (3.11+)

Raise and catch multiple unrelated exceptions at once.

```python
try:
    raise ExceptionGroup("multi", [
        ValueError("bad value"),
        KeyError("missing key"),
    ])
except* ValueError as eg:
    print("had a ValueError:", eg.exceptions)
except* KeyError as eg:
    print("had a KeyError:", eg.exceptions)
```

`except*` matches all exceptions of a type within a group, leaving the rest to re-raise. Useful with `asyncio.TaskGroup`.

---

## Context managers

The `with` block guarantees cleanup.

```python
with open("file.txt") as f:
    data = f.read()
# file is closed even if read() raises
```

Build your own:

```python
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.monotonic()
    yield
    print(f"took {time.monotonic() - start:.3f}s")

with timer():
    do_work()                # prints "took 0.123s" after work finishes
```

--

## contextlib helpers

```python
from contextlib import suppress, redirect_stdout, ExitStack
import io

# Swallow specific exceptions inside a block:
with suppress(FileNotFoundError):
    Path("missing.txt").unlink()    # no error if missing

# Redirect stdout to capture print output:
buf = io.StringIO()
with redirect_stdout(buf):
    print("hello")
buf.getvalue()                       # 'hello\n'

# Stack a variable number of context managers:
with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]
    # all files closed when the block exits
```

---

## Modules

A `.py` file is a module. Importing runs its top-level code once.

```python
import math
math.sqrt(2)                # 1.4142135623730951

from math import sqrt, pi
sqrt(2)                     # 1.4142135623730951
pi                          # 3.141592653589793

import math as m
m.sqrt(2)                   # 1.4142135623730951
```

---

## Packages

A directory with an `__init__.py` is a package.

```
myapp/
├── __init__.py        ← marks `myapp` as a package
├── models.py          ← import myapp.models
├── views.py
└── utils/
    ├── __init__.py    ← subpackage
    ├── strings.py     ← import myapp.utils.strings
    └── numbers.py
```

---

## How `import` works

![Python import system](/assets/diagrams/import-system.dot.svg)

`sys.modules` caches loaded modules — re-imports are essentially free.

---

## Standard library highlights

| Module | What it gives you |
| --- | --- |
| `collections` | `Counter`, `defaultdict`, `deque`, `namedtuple` |
| `itertools` | Lazy iterator combinators |
| `functools` | `lru_cache`, `partial`, `reduce`, `wraps` |
| `pathlib` | OO file paths |
| `re` | Regular expressions |
| `json` | JSON encode/decode |
| `datetime` | Dates, times, timezones |
| `typing` | Type hints |

---

## pathlib over os.path

```python
from pathlib import Path

config = Path.home() / ".config" / "app.toml"
config.exists()                  # True   (or False)
config.read_text()               # '<file contents as a str>'
config.write_text(new_data)      # 42     (number of characters written)

for py_file in Path("src").rglob("*.py"):
    print(py_file)               # src/app/main.py, src/app/utils.py, ...
```

---

## Reading & writing files

```python
# Text
content = Path("notes.txt").read_text(encoding="utf-8")  # str
Path("out.txt").write_text("hello")                      # 5  (chars written)

# Binary
data = Path("img.png").read_bytes()                      # bytes

# Streaming
with open("large.csv", "r") as f:
    for line in f:
        process(line)                                    # one line at a time
```

---

## The Zen of Python

```python
import this
```

> Beautiful is better than ugly.<br>
> Explicit is better than implicit.<br>
> Simple is better than complex.<br>
> Readability counts.<br>
> There should be one — and preferably only one — obvious way to do it.

— PEP 20, Tim Peters

---

## What's next

- **OOP** — classes, inheritance, ABCs, dataclasses, dunders
- **Metaprogramming** — decorators, descriptors, metaclasses
- **Refactoring & patterns**
