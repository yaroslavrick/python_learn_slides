---
title: PyTest
---

![Test results: three passing, one failing](/assets/images/topics/pytest.svg)
<!-- .element: class="title-illustration" -->

# PyTest

The de-facto Python testing framework.

---

## Why pytest?

- Plain `assert` statements — no `self.assertEqual` ceremony
- Powerful **fixtures** instead of `setUp` / `tearDown`
- **Parametrize** to run a test against many inputs
- A huge plugin ecosystem
- Detailed failure output (assert rewriting shows the values that failed)

Press **↓** for the alternatives and why pytest wins.

--

## The alternatives

| Framework | Status |
| --- | --- |
| **`unittest`** | Built into the stdlib (xUnit-style). Works, but verbose: `class TestX(unittest.TestCase)`, `self.assertEqual`, no parametrize, no fixtures. |
| **`nose`** / **`nose2`** | Older pytest-like runner. Largely abandoned (`nose` is unmaintained; `nose2` exists but lost mindshare). |
| **`doctest`** | Tests embedded in docstrings. Useful for documentation, not a primary test framework. |
| **`pytest`** | The de facto standard since ~2015. Active development, vast plugin ecosystem, lighter syntax. |

For new projects, **pick pytest**. For legacy `unittest` codebases, pytest **runs them as-is** — adoption is incremental.

---

## Your first test

```python
# tests/test_calculator.py
def add(a, b):
    return a + b

def test_add_returns_sum():
    assert add(2, 3) == 5

def test_add_with_negative():
    assert add(-1, 1) == 0
```

Run:

```bash
uv add --dev pytest
uv run pytest                  # auto-discovers tests/test_*.py
# ============================= 2 passed in 0.01s =============================
```

(Without uv: `pip install pytest && pytest`.)

No imports, no boilerplate, no `if __name__ == "__main__":`.

---

## Test discovery

By default, pytest collects:

- Files matching `test_*.py` or `*_test.py`
- Classes named `Test*` (without `__init__`)
- Functions named `test_*`

```
my-package/
├── pyproject.toml
├── src/
│   └── my_package/
│       └── core.py
└── tests/
    ├── conftest.py
    ├── test_core.py            # ✓ collected
    └── helpers.py              # ✗ not collected
```

---

## Test discovery — configuring

Override the defaults in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
```

---

## Running tests

```bash
uv run pytest                  # everything
uv run pytest tests/test_core.py
uv run pytest tests/test_core.py::test_add_returns_sum   # one test
uv run pytest -k "add and not neg"    # match by name expression
uv run pytest -m slow                 # by marker
uv run pytest -x                      # stop at first failure
uv run pytest --lf                    # last failed
uv run pytest --ff                    # failed first, then the rest
uv run pytest -v                      # verbose
uv run pytest -s                      # don't capture stdout (see prints)
```

The combo `uv run pytest -xvs path::name` is the everyday debugging incantation.

---

## Assert rewriting

pytest rewrites your `assert` statements to show **why** they failed.

```python
def test_user_dict():
    user = {"name": "Alice", "age": 30}
    assert user == {"name": "Bob", "age": 30}
```

```
>       assert user == {"name": "Bob", "age": 30}
E       AssertionError: assert {'age': 30, 'name': 'Alice'} == {'age': 30, 'name': 'Bob'}
E         Differing items:
E         {'name': 'Alice'} != {'name': 'Bob'}
```

Plain `assert` is genuinely enough.

---

## Fixtures — the basics

A **fixture** is a setup function whose return value is injected by name.

```python
import pytest

@pytest.fixture
def user():
    return {"name": "Alice", "age": 30}

def test_name(user):           # ← parameter name = fixture name
    assert user["name"] == "Alice"

def test_age(user):            # ← same fixture, fresh instance
    assert user["age"] == 30
```

pytest discovers fixtures by parameter name and runs them before the test.

--

## Fixtures — scope

Control how often a fixture runs.

```python
@pytest.fixture(scope="function")    # default — once per test
def user(): ...

@pytest.fixture(scope="class")       # once per test class
def db_table(): ...

@pytest.fixture(scope="module")      # once per .py file
def api_client(): ...

@pytest.fixture(scope="session")     # once per pytest run
def database(): ...
```

Use larger scopes for expensive setup (DB connections, web drivers). Don't share **mutable** state across tests with `session` scope unless you're confident about isolation.

--

## Fixtures — setup and teardown

`yield` splits a fixture into setup and teardown.

```python
@pytest.fixture
def db():
    conn = connect("sqlite://:memory:")
    create_schema(conn)
    yield conn               # ← test runs here
    conn.close()             # always runs, even on test failure
```

Equivalent to `unittest`'s `setUp`/`tearDown`, but per-fixture and composable.

--

## Fixtures — depending on fixtures

Fixtures can request other fixtures.

```python
@pytest.fixture
def db():
    return connect(":memory:")

@pytest.fixture
def alice(db):                 # ← db injected here
    return db.users.create(name="Alice")

def test_login(alice):         # ← alice injected; db happens transitively
    assert alice.login("pw")
```

Build small, focused fixtures. Compose them like Lego.

---

## conftest.py — shared fixtures

Fixtures defined in `conftest.py` are auto-discovered for every test in the same directory and subdirectories. This is pytest's main mechanism for sharing fixtures.

```
tests/
├── conftest.py            # ← fixtures available everywhere below
├── unit/
│   ├── conftest.py        # ← fixtures only for tests/unit/
│   └── test_user.py
└── integration/
    └── test_api.py
```

```python
# tests/conftest.py
import pytest

@pytest.fixture
def app():
    return create_app(testing=True)
```

No imports needed in test files — pytest finds them.

--

## Organizing many fixtures into modules

When you have lots of fixtures, putting them all in `conftest.py` gets noisy. Split them into per-domain modules:

```
tests/
├── conftest.py
└── fixtures/
    ├── __init__.py
    ├── users.py          # ← user-related fixtures
    ├── orders.py
    └── http.py
```

```python
# tests/fixtures/users.py
import pytest

@pytest.fixture
def alice(db):
    return db.users.create(name="Alice")
```

Press **↓** for how to register them.

--

## Registering fixture modules

In `conftest.py`, declare them as **pytest plugins**:

```python
# tests/conftest.py
pytest_plugins = [
    "tests.fixtures.users",
    "tests.fixtures.orders",
    "tests.fixtures.http",
]
```

That's it. Test files use the fixtures by name, just like before:

```python
def test_login(alice):           # `alice` from tests/fixtures/users.py
    assert alice.login("pw")
```

--

## Fixture file naming

Two rules:

- **Don't** end fixture files in `_test.py` or start with `test_` — pytest will try to collect tests from them.

  ✗ `tests/fixtures/user_fixture_test.py`

- **Do** use plain descriptive names:

  ✓ `tests/fixtures/users.py`
  ✓ `tests/fixtures/user_factories.py`

`conftest.py` is the only special filename. Everything else is just a regular Python module that you import or register as a plugin.

---

## Parametrize — same test, many inputs

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (-1, 1, 0),
    (0, 0, 0),
    (10, -5, 5),
])
def test_add(a, b, expected):
    assert a + b == expected
```

Output:

```
test_add[1-2-3]    PASSED
test_add[-1-1-0]   PASSED
test_add[0-0-0]    PASSED
test_add[10--5-5]  PASSED
```

Each row becomes a separate test — clear failures, clear coverage.

--

## Parametrize — IDs and stacking

```python
@pytest.mark.parametrize(
    "input,expected",
    [("hello", "HELLO"), ("World", "WORLD")],
    ids=["lowercase", "mixed-case"],         # nicer test names
)
def test_upper(input, expected):
    assert input.upper() == expected
```

Stack `parametrize` markers — pytest computes the cross-product:

```python
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_pairs(x, y): ...
# 4 tests: (1, 10), (1, 20), (2, 10), (2, 20)
```

---

## Markers

Tag tests for selection or special handling.

```python
@pytest.mark.slow
def test_full_pipeline(): ...

@pytest.mark.skip(reason="not implemented yet")
def test_future(): ...

@pytest.mark.skipif(sys.platform == "win32", reason="UNIX only")
def test_unix_only(): ...

@pytest.mark.xfail(reason="known bug #123")
def test_known_failure(): ...
```

--

## Markers — selection and registration

Run only marked tests:

```bash
uv run pytest -m slow                 # only the @slow ones
uv run pytest -m "not slow"           # everything except slow
uv run pytest -m "slow and not xfail" # boolean expressions
```

Register custom markers in `pyproject.toml` so typos fail loudly:

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow",
    "integration: hits external services",
]
```

With `--strict-markers` (recommended), `@pytest.mark.tpyo` is an error, not a silent no-op.

---

## Asserting exceptions

```python
import pytest

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0

def test_message():
    with pytest.raises(ValueError, match="must be positive"):
        sqrt(-1)

def test_capture():
    with pytest.raises(ValueError) as excinfo:
        sqrt(-1)
    assert "negative" in str(excinfo.value)
```

`pytest.raises` is the assert for "this should throw".

---

## Mocking

Use `unittest.mock` (stdlib) or `pytest-mock` (a thin wrapper).

```python
from unittest.mock import patch

def fetch_user(client, id):
    return client.get(f"/users/{id}").json()

@patch("my_pkg.api.client")
def test_fetch_user(mock_client):
    mock_client.get.return_value.json.return_value = {"id": 1, "name": "Alice"}
    assert fetch_user(mock_client, 1)["name"] == "Alice"
    mock_client.get.assert_called_once_with("/users/1")
```

--

## pytest-mock — a friendlier API

```python
def test_fetch_user(mocker):       # ← `mocker` fixture from pytest-mock
    client = mocker.Mock()
    client.get.return_value.json.return_value = {"id": 1, "name": "Alice"}

    result = fetch_user(client, 1)

    assert result == {"id": 1, "name": "Alice"}
    client.get.assert_called_once_with("/users/1")
```

`mocker` is auto-cleaned after each test — no `with patch(...)` blocks needed.

---

## tmp_path — real-file fixtures

pytest provides a per-test temp directory as `tmp_path` (a `pathlib.Path`).

```python
def test_writes_summary(tmp_path):
    out = tmp_path / "summary.txt"
    write_summary(out, ["a", "b", "c"])
    assert out.read_text() == "a\nb\nc\n"
```

The directory is unique per test and auto-cleaned. Use it for any test that touches the filesystem — never write to your repo root from a test.

---

## Plugin highlights

| Plugin | What it gives you |
| --- | --- |
| `pytest-cov` | Coverage reports (`pytest --cov=my_pkg`) |
| `pytest-mock` | The `mocker` fixture |
| `pytest-xdist` | Run tests in parallel (`pytest -n auto`) |
| `pytest-django` | Django integration (DB, client) |
| `pytest-asyncio` | Async test functions |
| `pytest-randomly` | Randomize test order — find hidden coupling |
| `pytest-snapshot` | Snapshot/golden testing |

```bash
uv add --dev pytest-cov pytest-mock pytest-xdist
uv run pytest --cov=src -n auto
```

---

## Configuring pytest

Everything in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --cov=src --cov-report=term-missing"
markers = [
    "slow: long-running test",
    "integration: hits external services",
]
filterwarnings = [
    "error",                       # treat warnings as errors
    "ignore::DeprecationWarning:third_party.*",
]
```

`--strict-markers` makes typos in `@pytest.mark.foo` fail loudly.

---

## A test-friendly mindset

- Tests are documentation that doesn't drift
- A failing test should tell you **what broke and why** in one screenful
- Write tests *before* the code when you can — even just one
- Slow tests get skipped — keep the unit suite under a few seconds
- Test behavior, not implementation: don't pin private attributes

Good test code is a feature.

---

## What's next

- **Static analysis** — ruff, mypy, pre-commit
- **Tooling deep dive** — uv, poetry, pyproject.toml
- **CI** — GitHub Actions for Python
