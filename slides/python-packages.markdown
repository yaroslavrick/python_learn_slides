---
title: Python packages — under the hood
---

![Stack of labeled package boxes](/assets/images/topics/python-packages.svg)
<!-- .element: class="title-illustration" -->

# Python packages

`pip`, virtual environments, `pyproject.toml`, publishing.

---

## What `uv` abstracts

In the previous deck we used `uv` for everything. Under the hood it's working with three things:

| Concept | Tool that owns it |
| --- | --- |
| Installing packages | **`pip`** |
| Isolating per-project deps | **`venv`** |
| Project metadata + deps spec | **`pyproject.toml`** |

This deck explains each. You'll see them when:

- Working in legacy projects without `uv`
- **Publishing your own package** to PyPI
- Debugging when something doesn't quite work

---

## `pip` — the package installer

Ships with Python. Installs from PyPI.

```bash
pip install requests             # latest
pip install requests==2.32.3     # exact version
pip install "requests>=2,<3"     # version range
pip uninstall requests
pip list                         # everything installed
pip show requests                # metadata for one package
```

In a uv project use `uv add requests` instead — it also updates `pyproject.toml` and `uv.lock`.

--

## pip — install variants

```bash
# From a Git repo
pip install git+https://github.com/owner/repo@v1.2.3

# From a local checkout, in editable mode
pip install -e ./my-lib

# Upgrade
pip install --upgrade requests

# From a requirements file
pip install -r requirements.txt

# With extras
pip install "fastapi[standard]"
```

--

## pip — listing and freezing

```bash
pip freeze
# → requirements.txt-friendly output:
# requests==2.32.3
# urllib3==2.5.0
# ...

pip freeze > requirements.txt    # snapshot
pip install -r requirements.txt  # restore on another machine

pip check                        # → "No broken requirements found."
```

`pip freeze` is **not** a real lock file — no hashes, doesn't distinguish direct vs transitive deps. For real locking use `uv lock` or `pip-tools`.

---

## Virtual environments

A **virtualenv** is an isolated Python — its own `pip`, its own packages — so projects don't fight over package versions.

```bash
python -m venv .venv             # → creates ./.venv/
source .venv/bin/activate        # macOS/Linux
.venv\Scripts\activate           # Windows

which python                     # /path/to/project/.venv/bin/python
pip install requests             # → installed into .venv only

deactivate                       # leave the environment
```

In a uv project, `.venv/` is created automatically by `uv add`/`uv sync`/`uv run` — you rarely interact with it directly.

--

## venv — `.gitignore` and editor setup

```gitignore
# .gitignore
.venv/
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.ruff_cache/
```

Modern editors auto-detect `.venv/`:

- **VS Code** — picks it up via the Python extension
- **PyCharm** — "Python Interpreter" → "Existing environment"
- **Neovim/coc** — usually via `pyrightconfig.json` or LSP autodetect

---

## `pyproject.toml`

The single configuration file for modern Python projects (PEP 518, 621). Both `uv` and `pip` read it.

```toml
[project]
name = "my-package"
version = "0.1.0"
description = "An example package"
readme = "README.md"
requires-python = ">=3.12"
authors = [{ name = "Alice", email = "alice@example.com" }]
license = "MIT"
dependencies = [
    "requests>=2.32",
    "click>=8.1",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

`uv add` writes into `[project.dependencies]` for you.

--

## pyproject.toml — optional dependency groups

Group extras for different use cases (`dev`, `test`, `docs`, ...).

```toml
[project.optional-dependencies]
dev = ["pytest>=8", "ruff", "mypy"]
docs = ["mkdocs", "mkdocs-material"]

# PEP 735 (modern, dev-only — what `uv add --dev` writes):
[dependency-groups]
dev = ["pytest>=8", "ruff", "mypy"]
```

Install:

```bash
pip install ".[dev]"             # core + 'dev' extra
pip install -e ".[dev,docs]"     # editable, multiple extras

# uv equivalent:
uv sync                           # all groups by default
uv sync --no-dev                  # skip dev
```

--

## pyproject.toml — entry points

Expose CLI commands when the package is installed.

```toml
[project.scripts]
my-tool = "my_package.cli:main"
```

After install:

```bash
my-tool --help                   # runs my_package.cli.main()
```

The user gets a real binary on `PATH` — no `python -m my_package` needed.

---

## Project layout — `src/`

```
my-package/
├── pyproject.toml
├── README.md
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── core.py
│       └── cli.py
└── tests/
    ├── __init__.py
    └── test_core.py
```

The `src/` layout prevents `import my_package` from accidentally picking up your working tree instead of the installed copy — making tests honest about what users will get.

`uv init` uses the `src/` layout by default.

---

## Building distributions

Two artifact types:

| Type | Description |
| --- | --- |
| **sdist** (`.tar.gz`) | Source distribution — code + metadata |
| **wheel** (`.whl`) | Pre-built binary — what `pip install` actually wants |

```bash
# uv way
uv build                         # → dist/*.tar.gz and dist/*.whl

# without uv
pip install build
python -m build
```

The build backend (declared in `[build-system]`) does the work — `hatchling`, `setuptools`, `flit`, `poetry-core`, ...

---

## Publishing to PyPI

```bash
# uv way
uv publish                       # uses ~/.pypirc or UV_PUBLISH_TOKEN

# without uv
pip install twine
twine upload dist/*              # → asks for username/password (or API token)
```

After publish:

```bash
pip install my-package
# or
uv add my-package
```

Test on **TestPyPI** first:

```bash
uv publish --publish-url https://test.pypi.org/legacy/
```

--

## Trusted Publishing (no tokens)

PyPI supports **OpenID Connect** with GitHub Actions — no API tokens to leak.

```yaml
# .github/workflows/publish.yml
name: Publish
on: { push: { tags: ['v*'] } }
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions: { id-token: write }    # required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

Configure the project on PyPI under "Publishing" with your GitHub repo — that's the whole setup.

---

## Why know any of this?

`uv` is the easy path. But:

- **You still write `pyproject.toml`** — it's the source of truth
- **Publishing libraries** — `uv publish` is great, but understanding sdists, wheels, and build backends helps you debug
- **Other people's projects** — half the open-source world still uses plain `pip` + `venv`
- **Constrained environments** — corporate networks, air-gapped CI, embedded contexts

The abstraction is worth using. The foundations are worth understanding.

---

## What's next

- **Task runners** — Make, `invoke`, `click`, `typer`
- **PyTest** — testing
- **Static analysis** — `ruff`, `mypy`, `pre-commit`
