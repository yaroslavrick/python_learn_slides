---
title: Project setup with uv
---

![A toolbox with a wrench and a checkmark](/assets/images/topics/python-tooling.svg)
<!-- .element: class="title-illustration" -->

# Project setup with `uv`

The modern, fast Python toolchain.

---

## Why one tool for everything?

Historically you needed:

- `pyenv` / `asdf` — install Python versions
- `python -m venv` — make virtual environments
- `pip` — install packages
- `pip-tools` — lock dependencies
- `pipx` — install CLI tools globally
- `build` + `twine` — build and publish packages

`uv` does **all of that**, ~10–100× faster, in one binary.

---

## Recap from Python basics

The **Python basics** deck already walked through:

- Installing `uv` (Step 1)
- Installing Python via `uv python install` (Step 2)
- Scaffolding a project with `uv init` (Step 3)
- Adding deps with `uv add` and running with `uv run` (Step 4)

This deck goes **deeper** — the full pyproject layout, `uv add` variants, `uv run` options, `uv tool`, `uv pip`, `uv sync`, the lock file, and the alternatives uv replaces.

---

## What `pyproject.toml` looks like

`uv init` produces this:

```toml
[project]
name = "my-app"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

`uv add` writes into `[project.dependencies]`. `uv add --dev` writes into `[dependency-groups.dev]` (PEP 735).

`.venv/` is **not** created until `uv` first needs it (e.g., `uv add` or `uv run`).

---

## What `uv add` actually does

Behind one command:

```bash
uv add requests
```

Four steps, atomically:

1. **Resolves** a compatible version against your `requires-python` and existing deps
2. **Adds** the dep under `[project.dependencies]` in `pyproject.toml`
3. **Updates** `uv.lock` (with hashes for every transitive dep)
4. **Installs** into `.venv/` — a **virtual environment**: an isolated folder holding the project's Python interpreter and its installed packages, so projects don't share or fight over versions

If any step fails, nothing changes. Same machine, same `uv.lock` → byte-identical environment.

--

## `uv add` — useful variants

```bash
# From a Git repo
uv add "git+https://github.com/owner/repo@v1.2.3"

# Editable install of a local sibling project
uv add --editable ../my-lib

# With extras (optional dependency groups of the package)
uv add "fastapi[standard]"

# Remove
uv remove requests

# Upgrade everything (re-resolve)
uv lock --upgrade
uv sync
```

---

## Run things with `uv run`

`uv run` ensures dependencies are synced, then executes the command **inside** the project's venv.

```bash
uv run python main.py           # → Hello from my-app!
uv run python -m my_app
uv run pytest
uv run ruff check . --fix
uv run mypy src/
```

You never manually `source .venv/bin/activate`. uv handles that.

--

## `uv run` — handy options

```bash
# Inline-deps script: install just for this run, no project needed
uv run --with httpx --with rich ./quick_script.py

# Use a different Python for this command
uv run --python 3.11 pytest

# Run in a specific directory
uv run --directory ./packages/api pytest
```

Inline deps are great for one-off scripts: declare them at the top of the file and `uv run` resolves them.

```python
# scrape.py
# /// script
# requires-python = ">=3.12"
# dependencies = ["httpx", "rich"]
# ///
import httpx; from rich import print
print(httpx.get("https://httpbin.org/get").json())
```

```bash
uv run scrape.py                # uv resolves httpx + rich on the fly
```

---

## `uv tool` — global CLI tools

Replaces `pipx`. Install Python tools to use *globally*, isolated from any project.

```bash
uv tool install ruff             # → `ruff` on PATH, isolated venv
uv tool install ipython
uv tool list                     # what's installed
uv tool upgrade --all
uv tool uninstall ruff

# One-off run without installing
uv tool run ruff check .         # alias: `uvx ruff check .`
```

Use `uv add --dev ruff` when ruff is part of a project; `uv tool install ruff` when you just want a global `ruff` everywhere.

---

## `uv pip` — the legacy bridge

`uv pip` is a drop-in replacement for `pip` — same flags, much faster — for cases where the modern `uv add` workflow doesn't fit.

```bash
uv pip install rich              # install into the active venv (no pyproject update)
uv pip install -r requirements.txt
uv pip list
uv pip freeze > requirements.txt
uv pip compile pyproject.toml -o requirements.txt    # like pip-tools
```

Use this for:

- Legacy projects still on `requirements.txt`
- Quick installs into an existing venv you don't own
- Tooling that expects `pip`-style commands

In a uv-managed project, prefer `uv add` instead — it keeps `pyproject.toml` and `uv.lock` in sync.

---

## `uv sync` — install exactly what's locked

```bash
uv sync                          # → install everything from uv.lock into .venv/
uv sync --frozen                 # → CI: don't re-resolve, fail if lock is stale
uv sync --no-dev                 # → skip dev deps
uv sync --extra docs             # → include the `docs` extra
```

CI command of choice:

```yaml
- run: uv sync --frozen
- run: uv run pytest
```

---

## The `uv.lock` file

```toml
# uv.lock (excerpt)
[[package]]
name = "requests"
version = "2.32.3"
source = { registry = "https://pypi.org/simple" }
sdist = { url = "...", hash = "sha256:..." }
wheels = [{ url = "...", hash = "sha256:..." }]
```

- Records every transitive dependency, with hashes
- Pins exact versions for reproducible installs
- Cross-platform (resolves for Linux/macOS/Windows in one file)

--

## `uv.lock` — commit it?

| Project type | Lock file? |
| --- | --- |
| **Application** (web service, CLI, internal tool) | **Commit** `uv.lock` |
| **Library** published to PyPI | **Don't commit** — users get fresh resolution |

For an application, `uv.lock` is your reproducibility guarantee: same lock file → byte-identical environment in dev, CI, and production.

For a library, your *consumers* resolve dependencies against their own constraints — your lock file would be ignored anyway, and might mislead contributors into thinking it's authoritative.

---

## Alternatives — quick tour

Press **↓** to see what uv replaced.

--

## `pip` + `venv` (built-in)

The lowest common denominator — always available, slowest.

```bash
python -m venv .venv             # create env
source .venv/bin/activate        # activate it
pip install requests             # install
pip freeze > requirements.txt    # snapshot
deactivate                       # leave
```

No locking with hashes by default. Fine for very small things; everything else benefits from a higher-level tool.

--

## `poetry`

The established, all-in-one alternative.

```bash
pipx install poetry
poetry new my-app
cd my-app
poetry add requests
poetry add --group dev pytest
poetry install
poetry run pytest
poetry build
poetry publish
```

Pros: mature, well-known.
Cons: slower than uv, its own pyproject dialect (though recent versions support standard `[project]`).

--

## `pip-tools`

Add locking to `pip` without changing much else.

```bash
pip install pip-tools
pip-compile pyproject.toml -o requirements.txt
pip-sync requirements.txt
```

Minimalist, predictable. A reasonable choice if your team is allergic to "yet another tool".

---

## Recap — daily commands

| Task | Command |
| --- | --- |
| New project | `uv init my-app` |
| Pin Python version | `uv python pin 3.13` |
| Add a dep | `uv add requests` |
| Add a dev dep | `uv add --dev pytest` |
| Remove a dep | `uv remove requests` |
| Run something | `uv run pytest` |
| Sync from lock | `uv sync --frozen` |
| Install a global tool | `uv tool install ruff` |
| Run a global tool once | `uvx ruff check .` |
| Drop-in pip | `uv pip install <pkg>` |

Memorize three: `uv add`, `uv run`, `uv sync --frozen`. The rest you'll learn as needed.

---

## What's next

- **Python packages** — what `uv` is doing under the hood: pip, venv, `pyproject.toml`, publishing
- **Task runners** — Make, `invoke`, `click`, `typer`
- **PyTest** — testing
