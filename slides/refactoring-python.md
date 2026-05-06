---
title: Refactoring Python
---

![Tangled code becoming clean](/assets/images/topics/refactoring.svg)
<!-- .element: class="title-illustration" -->

# Refactoring Python

Improving the structure of code without changing its behavior.

---

## Why refactor?

- Make code easier to **read** and **change** later
<!-- .element: class="fragment" -->
- Surface hidden coupling and duplication
<!-- .element: class="fragment" -->
- Set the stage for new features
<!-- .element: class="fragment" -->
- Pay down debt before it accrues interest
<!-- .element: class="fragment" -->

> "Make the change easy, then make the easy change." — Kent Beck

---

## The two hats

> "When using refactoring, you divide your time between two distinct activities: adding function and refactoring. When you add function, you shouldn't be changing existing code; you're just adding new capabilities. When you refactor, you make a resolution not to add any function; you only restructure the code." — Martin Fowler

Never wear both hats at once.

---

## Tests are the safety net

You can't safely refactor without tests. Before changing anything:

1. Write tests for the behavior you're about to refactor
2. Make sure they pass
3. Refactor in small steps
4. Run tests after each step

Without tests, refactoring is just rewriting and hoping.

---

## Code smells

Surface symptoms of deeper design problems. We'll work through the common ones.

- Long function
- Long parameter list
- Magic numbers
- Duplicated code
- Feature envy
- Data clumps
- Primitive obsession
- Conditional complexity

---

## Smell: Long function

Probably doing more than one thing.

```python
def process_order(order):
    # validate
    if order.total < 0: raise ValueError(...)
    if not order.items: raise ValueError(...)
    # compute discount
    discount = 0
    if order.coupon:
        discount = order.subtotal * 0.1
    # charge
    stripe.charge(order.user, order.total - discount)
    # email
    send_email(order.user.email, "Order confirmed", ...)
    # log
    logger.info(f"order {order.id} processed")
```

---

## Refactor: extract function

Break it apart by responsibility:

```python
def process_order(order):
    validate(order)
    discount = compute_discount(order)
    charge(order, discount)
    notify(order)
    logger.info(f"order {order.id} processed")
```

Each helper is small, named, and individually testable.

---

## Smell: Long parameter list

```python
def create_user(name, email, age, country, role,
                is_active, is_admin, signup_source, ...):
    ...
```

If a function takes more than ~4 arguments, the arguments probably travel together.

---

## Refactor: introduce parameter object

```python
@dataclass
class UserSpec:
    name: str
    email: str
    age: int
    country: str
    role: str = "user"
    is_active: bool = True
    is_admin: bool = False
    signup_source: str = "web"

def create_user(spec: UserSpec) -> User:
    ...
```

The dataclass makes the relationships explicit and gives you a single thing to pass around.

---

## Smell: Magic numbers

```python
if user.age < 18:
    return "denied"
total = price * 0.07         # what is 0.07?
sleep(86400)                 # what's that in?
```

A bare number tells you nothing about intent.

---

## Refactor: name the constant

```python
LEGAL_AGE = 18
SALES_TAX = 0.07
ONE_DAY_SECONDS = 24 * 60 * 60

if user.age < LEGAL_AGE: ...
total = price * SALES_TAX
sleep(ONE_DAY_SECONDS)
```

Constants live at the top of the module or in a small `constants.py`.

---

## Smell: Duplicated code

```python
def hash_password(p):
    return hashlib.sha256(p.encode()).hexdigest()

def hash_token(t):
    return hashlib.sha256(t.encode()).hexdigest()
```

Or, more insidious:

```python
# in views.py
total = sum(item.price * item.qty for item in items)

# in reports.py
total = sum(item.price * item.qty for item in items)
```

---

## Refactor: extract function / module

```python
# common.py
def sha256_hex(s: str) -> str:
    return hashlib.sha256(s.encode()).hexdigest()

# orders.py
def order_total(items: Iterable[Item]) -> Decimal:
    return sum(item.price * item.qty for item in items)
```

The rule of three: duplicate once, extract on the third.

---

## Smell: Feature envy

A method that uses another object's data more than its own.

```python
class Order:
    def shipping_label(self):
        return (
            f"{self.customer.name}\n"
            f"{self.customer.address.line1}\n"
            f"{self.customer.address.city}, "
            f"{self.customer.address.zip}"
        )
```

`Order` is rummaging through `customer` and `address`.

---

## Refactor: move method

```python
class Address:
    def formatted(self) -> str:
        return f"{self.line1}\n{self.city}, {self.zip}"

class Customer:
    def shipping_label(self) -> str:
        return f"{self.name}\n{self.address.formatted()}"

class Order:
    def shipping_label(self):
        return self.customer.shipping_label()
```

Each class owns its own data.

---

## Smell: Data clumps

The same group of fields appearing together everywhere.

```python
def book_flight(name, email, phone, *, departure, arrival): ...
def book_hotel(name, email, phone, *, hotel, dates): ...
def book_car(name, email, phone, *, car_type): ...
```

Three fields glued together: extract them.

---

## Refactor: extract class

```python
@dataclass
class ContactInfo:
    name: str
    email: str
    phone: str

def book_flight(contact: ContactInfo, *, departure, arrival): ...
def book_hotel(contact: ContactInfo, *, hotel, dates): ...
```

---

## Smell: Primitive obsession

Using strings or ints for things that have **meaning** beyond their value.

```python
def transfer(amount: float, currency: str): ...
def send_email(to: str): ...

transfer(100, "USD")
transfer(100, "USDD")        # typo, no error until runtime
send_email("not-an-email")   # also no error
```

---

## Refactor: introduce a domain type

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: Currency       # an Enum

@dataclass(frozen=True)
class EmailAddress:
    value: str
    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError(self.value)

def transfer(amount: Money): ...
def send_email(to: EmailAddress): ...
```

Now invalid states are unrepresentable.

---

## Smell: Conditional complexity

Nested conditionals are a sign of mixed abstraction levels.

```python
def discount(user, order):
    if user.is_premium:
        if order.total > 100:
            return 0.20
        else:
            return 0.10
    else:
        if order.total > 200:
            return 0.05
        else:
            return 0.0
```

---

## Refactor: early returns / flatten

```python
def discount(user, order):
    if user.is_premium and order.total > 100: return 0.20
    if user.is_premium:                       return 0.10
    if order.total > 200:                     return 0.05
    return 0.0
```

Each line is now an isolated rule. The pyramid is gone.

---

## Refactor: replace conditional with polymorphism

When a `match`/`if/elif` switches on **type**, lift the branches into the types themselves.

```python
# Before
def area(shape):
    if shape.kind == "circle":
        return 3.14 * shape.r ** 2
    elif shape.kind == "rect":
        return shape.w * shape.h
```

```python
# After
class Circle:    def area(self): return 3.14 * self.r ** 2
class Rectangle: def area(self): return self.w * self.h

def area(shape): return shape.area()
```

---

## Refactor: replace loop with comprehension

```python
# Before
result = []
for item in items:
    if item.active:
        result.append(item.name.upper())
```

```python
# After
result = [item.name.upper() for item in items if item.active]
```

Comprehensions communicate "I'm building a list" without the bookkeeping. Don't go further than that — comprehensions with side effects, multiple `if`s, or wrapping `try` blocks should stay as loops.

---

## Refactor: replace temp with query

```python
# Before
def total(items):
    base = sum(i.price for i in items)
    if base > 100: shipping = 0
    else:          shipping = 10
    return base + shipping
```

```python
# After
def base_price(items):
    return sum(i.price for i in items)

def shipping(items):
    return 0 if base_price(items) > 100 else 10

def total(items):
    return base_price(items) + shipping(items)
```

Each helper is reusable and individually testable.

---

## Comments as a smell

Comments often explain code that should explain itself.

```python
# add 7 days to the date
deadline = date + timedelta(days=7)
```

vs.

```python
GRACE_PERIOD = timedelta(days=7)
deadline = date + GRACE_PERIOD
```

Keep the comments that explain **why** (a non-obvious constraint, a workaround). Drop the ones that narrate **what**.

---

## SOLID — overview

| Letter | Principle |
| --- | --- |
| **S** | Single Responsibility |
| **O** | Open/Closed |
| **L** | Liskov Substitution |
| **I** | Interface Segregation |
| **D** | Dependency Inversion |

---

## SRP — Single Responsibility

A class should have **one reason to change**.

```python
# Bad: ReportGenerator does three things
class ReportGenerator:
    def fetch_data(self): ...
    def render_html(self): ...
    def email_to_user(self): ...
```

```python
# Better: split by reason-to-change
class ReportData:    def fetch(self): ...
class ReportRenderer: def to_html(self, data): ...
class Mailer:         def send(self, html, user): ...
```

---

## OCP — Open/Closed

Open to extension, closed to modification.

```python
# Adding a new payment method should not require editing this:
def charge(order):
    if order.method == "card":   stripe.charge(...)
    elif order.method == "paypal": paypal.charge(...)
    elif order.method == "crypto": btc.charge(...)
```

```python
# Better: a Protocol, plug-in implementations:
class PaymentGateway(Protocol):
    def charge(self, order) -> None: ...

def charge(order, gateway: PaymentGateway):
    gateway.charge(order)
```

New methods = new classes. No edits needed.

---

## LSP — Liskov Substitution

Subtypes must honor the parent's contract — including invariants and exceptions.

```python
class Bird:
    def fly(self): ...

class Penguin(Bird):
    def fly(self): raise NotImplementedError    # ← LSP violation
```

If subclasses can't keep the contract, the base class is wrong.

---

## ISP — Interface Segregation

Small, focused interfaces beat one fat one.

```python
# Bad
class Worker(Protocol):
    def work(self): ...
    def eat(self): ...
    def sleep(self): ...
```

```python
# Better
class Workable(Protocol): def work(self): ...
class Eatable(Protocol):  def eat(self): ...
class Sleepable(Protocol): def sleep(self): ...
```

Robots can be `Workable` without pretending to eat.

---

## DIP — Dependency Inversion

High-level modules depend on **abstractions**, not concretes.

```python
# Bad
class OrderService:
    def __init__(self):
        self.db = PostgresDB()           # hard-wired

# Better
class OrderService:
    def __init__(self, db: DataStore):
        self.db = db                     # injected
```

Tests can pass an in-memory `DataStore`. Production passes a Postgres-backed one. The service doesn't know or care.

---

## Type hints as refactoring guidance

Adding types to a legacy module surfaces:

- Implicit `Optional`s (variables that can be `None`)
- Functions that return more than one type
- Sequences treated as if they were unique
- Dict-shaped objects masquerading as records

Fixing those is half the refactoring already.

---

## Tools

| Tool | What it does |
| --- | --- |
| `ruff` | Lint + autofix common smells |
| `ruff format` | Format like Black |
| `mypy` / `pyright` | Static type-check |
| `refurb` | Suggest modern idioms |
| `vulture` | Find dead code |
| `radon` | Cyclomatic complexity |

Run them in CI, fix as you touch code.

---

## When NOT to refactor

- The code works, isn't going to change, and tests are thin
<!-- .element: class="fragment" -->
- You don't have a deadline buffer
<!-- .element: class="fragment" -->
- You're rewriting "for the future" — a future that may not come
<!-- .element: class="fragment" -->
- You haven't agreed on what "better" means with your team
<!-- .element: class="fragment" -->

> Refactoring is investing. Make sure the return is real.

---

## A workflow

1. Find a piece of code you'll touch this week
2. Add tests around it
3. Make one small refactor — extract a function, rename, flatten
4. Run tests
5. Commit
6. Repeat

Tiny, frequent steps. Never a "big refactor PR".

---

## What's next

- **Patterns** — when classical patterns help, and when Python makes them disappear
