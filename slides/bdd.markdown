---
title: BDD
---

![Given-When-Then scenario](/assets/images/topics/bdd.svg)
<!-- .element: class="title-illustration" -->

# Behavior-driven development

`pytest-bdd` and `behave` against a Django app.

---

## What BDD adds

Test scenarios written in **plain English**, mapped to Python step definitions:

```gherkin
Feature: Buying a book

  Scenario: User can add a book to their cart
    Given an authenticated user "alice"
    And a book "Crime and Punishment" exists
    When alice adds "Crime and Punishment" to her cart
    Then her cart contains 1 item
    And the total is 12.99
```

The scenario reads like a spec a non-developer can review. The steps are tested by real Python.

---

## When BDD pays off

- Stakeholders / PMs want to **read** the test suite
- Acceptance criteria are negotiated in tickets and you want them to *be* the tests
- Many similar scenarios share a few setup / verification steps

When all your tests are unit tests, BDD adds ceremony for little gain. Use BDD where the scenarios genuinely communicate.

---

## `pytest-bdd` — modern Python BDD

```bash
uv add --dev pytest-bdd pytest-django
```

Directory layout:

```
tests/
├── conftest.py
├── features/
│   └── cart.feature
└── step_defs/
    └── test_cart_steps.py
```

The `_test.py` filename is required for pytest collection.

---

## A `.feature` file

```gherkin
# tests/features/cart.feature
Feature: Shopping cart

  Background:
    Given an empty cart for "alice"

  Scenario: Adding a single item
    When alice adds "Crime and Punishment" priced 12.99 to her cart
    Then her cart contains 1 item
    And the total is 12.99

  Scenario: Adding two of the same item
    When alice adds "Dune" priced 9.99 to her cart 2 times
    Then her cart contains 2 items
    And the total is 19.98
```

Gherkin keywords: `Given`, `When`, `Then`, `And`, `But`, `Background`, `Scenario`, `Scenario Outline`.

---

## Step definitions

```python
# tests/step_defs/test_cart_steps.py
from pytest_bdd import scenarios, given, when, then, parsers
from decimal import Decimal
from accounts.models import User
from shop.models import Cart, Book

scenarios("../features/cart.feature")

@given(parsers.parse('an empty cart for "{username}"'), target_fixture="cart")
def empty_cart(db, username):
    user = User.objects.create_user(username=username)
    return Cart.objects.create(user=user)

@when(parsers.parse('{username} adds "{title}" priced {price:f} to her cart'))
def add_item(cart, username, title, price):
    book = Book.objects.create(title=title, price=Decimal(str(price)))
    cart.items.create(book=book, qty=1)

@then(parsers.parse("her cart contains {n:d} item"))
@then(parsers.parse("her cart contains {n:d} items"))
def cart_count(cart, n):
    assert cart.items.count() == n
```

`scenarios("...")` is the magic line — it discovers every scenario in the feature file and turns each into a pytest test.

---

## Step parsers

`parsers.parse` matches `{name}` placeholders by type:

| Pattern | Captures |
| --- | --- |
| `{x}` | str (default) |
| `{x:d}` | int |
| `{x:f}` | float |
| `{x:w}` | word (no spaces) |

For more complex matches, `parsers.cfparse` (case-insensitive) and `parsers.re` (regex).

---

## Sharing state — fixtures

Pass state between steps via pytest fixtures and `target_fixture`:

```python
@given(parsers.parse('a user "{username}"'), target_fixture="user")
def given_user(db, username):
    return User.objects.create_user(username=username)

@when(parsers.parse('{username} logs in'))
def login_step(client, user):       # `user` injected from previous step
    client.force_login(user)
```

Each scenario gets fresh fixtures. `db` (from `pytest-django`) wraps each scenario in a transaction that's rolled back.

---

## Scenario outlines — data tables

```gherkin
Scenario Outline: Adding multiple items
  Given an empty cart for "alice"
  When alice adds "<title>" priced <price> to her cart
  Then the total is <total>

  Examples:
    | title           | price | total |
    | Crime           | 12.99 | 12.99 |
    | Dune            | 9.99  | 9.99  |
    | Cervantes' Don  | 14.50 | 14.50 |
```

One scenario, three test cases. Same step definitions handle all three.

---

## `behave` — the original

`pytest-bdd` integrates with pytest. `behave` is a standalone runner closer to Cucumber.

```bash
uv add --dev behave behave-django
uv run python manage.py behave
```

Layout:

```
features/
├── steps/
│   └── cart.py
└── cart.feature
```

```python
# features/steps/cart.py
from behave import given, when, then

@given('an empty cart for "{username}"')
def step_impl(context, username):
    context.user = User.objects.create_user(username=username)
    context.cart = Cart.objects.create(user=context.user)
```

`context` is the shared scratchpad across steps.

`pytest-bdd` wins for most Django projects (you already have pytest); `behave` if you have a non-Python team that wants to read scenarios.

---

## What BDD doesn't replace

- Unit tests for **business logic** — services, queryset methods, validators. Faster, more focused.
- **Property-based** tests for tricky algorithms — `hypothesis`.
- Performance tests, security tests, browser end-to-end tests.

BDD is one layer in a balanced suite. Don't make every test a feature file.

---

## Anti-patterns

- **One step per assert** — `Then the response status is 200` followed by 30 micro-steps. Steps should be *meaningful* business operations.
- **Shared mutable global state** between scenarios — kills the rollback isolation pytest-django gives you.
- **Stuffing implementation details in the .feature** — "When the SQL `SELECT * FROM ...` runs". The feature is for humans; keep DB details in the step definitions.
- **Treating BDD as the unit test framework** — slow, ceremonious, hard to debug.

---

## Workflow

1. PM writes acceptance criteria as Gherkin
2. Dev pairs with PM to refine wording
3. Dev implements step definitions and the feature
4. PM reviews failing → passing scenario in CI
5. The same `.feature` becomes documentation

When the cycle works, the .feature file is the **single source of truth** for what the feature does — better than tickets that drift.

---

## What's next

- **Celery** — testing async tasks
- **DRF** — API tests, often a good complement to BDD
