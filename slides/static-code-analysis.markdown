---
title: Static code analysis
---

![Magnifier inspecting code lines](/assets/images/topics/static-code-analysis.svg)
<!-- .element: class="title-illustration" -->

# Static code analysis

`ruff`, `mypy`, `pre-commit`. Catch bugs before they run.

---

## What and why?

| Tool category | Catches | Tools |
| --- | --- | --- |
| **Linter** | Style, common bugs, dead code | `ruff` |
| **Formatter** | Whitespace, quotes, line wraps | `ruff format`, `black` |
| **Type checker** | Type mismatches, `None` handling | `mypy`, `pyright` |
| **Hook runner** | Run all of the above on commit | `pre-commit` |

Each catches a different class of mistake. Run all four.

---

## ruff — the modern multitool

`ruff` is a Rust-based linter and formatter that **replaces** half a dozen older tools (`flake8`, `pylint`, `isort`, `pyupgrade`, `pydocstyle`, `bandit`, ...).

```bash
uv add --dev ruff

uv run ruff check .            # lint
uv run ruff check --fix .      # lint + autofix what's safe
uv run ruff format .           # format (Black-compatible)
uv run ruff format --check .   # CI mode: fail without writing
```

It's fast — typically **10–100×** faster than the tools it replaces.

--

## ruff — selecting rules

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = [
    "E", "W",   # pycodestyle (errors, warnings)
    "F",        # pyflakes (unused imports, undefined names)
    "I",        # isort (import order)
    "B",        # flake8-bugbear (likely bugs)
    "UP",       # pyupgrade (modernize syntax)
    "N", "SIM", "C4",   # naming, simplify, comprehensions
]
ignore = ["E501"]              # let formatter handle line length
```

Hundreds of rules — enable what you want, ignore the rest.

--

## ruff — per-file ignores

Different parts of the codebase have different rules. Override per path:

```toml
[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]          # allow `assert` in tests
"__init__.py" = ["F401"]       # re-exports without "unused" complaints
"scripts/**" = ["T201"]        # allow plain print() in scripts
```

Pattern syntax is the standard glob. Rule codes match what `ruff check` reports.

--

## ruff — formatting

`ruff format` produces output **byte-identical to Black** for the vast majority of code, with a fraction of the runtime.

```bash
uv run ruff format src/ tests/  # rewrite files in place
uv run ruff format --check src/ # exit 1 if changes needed (CI)
uv run ruff format --diff src/  # preview changes
```

If you already use Black, swap the binary — same configuration philosophy ("opinionated, no knobs"), nothing else changes.

---

## mypy — type checking

```python
def add(a: int, b: int) -> int:
    return a + b

add(2, 3)              # OK
add("2", 3)            # error: Argument 1 has incompatible type "str"; expected "int"
```

```bash
uv add --dev mypy
uv run mypy src/
```

Type hints are documentation that gets verified. Refactoring on a typed codebase is dramatically safer.

--

## mypy — going strict

Defaults are too permissive. Tighten them.

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true                  # turns on all the recommended checks
warn_unused_ignores = true
warn_redundant_casts = true
exclude = ["build/", "dist/"]

[[tool.mypy.overrides]]
module = ["legacy_module.*"]   # silence one stubborn area
ignore_errors = true
```

`strict` is shorthand for: `disallow_untyped_defs`, `disallow_untyped_calls`, `no_implicit_optional`, `warn_return_any`, ...

Turn it on for new code; relax per-module for legacy.

--

## mypy — gradual typing

You don't have to type everything at once.

```python
from typing import Any

def process(data: Any) -> Any:    # explicit "I'll type this later"
    ...

# In a config or per-file:
# type: ignore[call-arg]          # silence one specific check
```

The point is to type **boundaries** first (public APIs, models, return values) and let the rest follow.

---

## pyright (the alternative)

Microsoft's TypeScript-style checker, written in TS itself. The engine behind Pylance in VS Code.

| | `mypy` | `pyright` |
| --- | --- | --- |
| Author | Python community | Microsoft |
| Speed | Fast | Faster |
| Default in CI | Common | Common |
| Editor integration | OK | Excellent (Pylance) |
| Strictness profiles | Manual | Built-in modes |

Either is fine. Many teams run `pyright` in the editor and `mypy` in CI.

---

## pre-commit — gluing it all together

`pre-commit` runs your linters/formatters on the files you're about to commit. No commit is allowed if anything fails.

```bash
uv tool install pre-commit     # global tool — replaces pipx install
pre-commit install             # registers .git/hooks/pre-commit
```

Configuration lives in `.pre-commit-config.yaml` at the project root.

--

## pre-commit — config example

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.13.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.18.0
    hooks:
      - id: mypy
        additional_dependencies: ["pydantic", "types-requests"]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

--

## pre-commit — workflow

```bash
git add src/foo.py
git commit -m "..."
# ruff............................Passed
# ruff-format.....................Passed
# mypy............................Failed   ← commit aborted
# - error: Argument 1 to "save" has type "str"; expected "Path"
```

The commit is **blocked** until everything passes — you can't push broken code by accident.

```bash
pre-commit run --all-files     # check the whole repo
pre-commit autoupdate          # bump hook versions
```

--

## pre-commit — CI

Run the same hooks in CI so unhooked teammates can't slip past.

```yaml
# .github/workflows/lint.yml
jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - uses: pre-commit/action@v3.0.1
```

The CI run is the source of truth; the local hook is a fast feedback loop.

---

## A tight Python toolchain

Everything in one `pyproject.toml`:

```toml
[project]
requires-python = ">=3.12"

[dependency-groups]
dev = ["pytest>=8", "pytest-cov", "ruff", "mypy", "pre-commit"]
```

Press **↓** for the lint, type-check, and test config.

--

## Lint and format config

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "B", "UP", "N", "SIM", "C4"]
```

--

## Type-check and test config

```toml
[tool.mypy]
python_version = "3.12"
strict = true

[tool.pytest.ini_options]
addopts = "-ra --strict-markers --cov=src"
testpaths = ["tests"]
```

Modern Python projects keep all of it here. No `setup.cfg`, no `mypy.ini`, no `pytest.ini` — one file, one source of truth.

---

## What's next

- **Tooling deep dive** — uv vs poetry, lock files, build backends
- **CI** — GitHub Actions for Python
