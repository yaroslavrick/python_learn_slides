---
title: Task runners
---

![A list of named tasks with checkmarks](/assets/images/topics/task-runners.svg)
<!-- .element: class="title-illustration" -->

# Task runners

Make, `invoke`, `click`, `typer` — running common project commands.

---

## Why a task runner?

Every project ends up with a handful of incantations:

```bash
uv sync                                          # install everything
uv run pytest -xvs tests/                        # run tests
uv run ruff format . && uv run ruff check --fix . # format and lint
uv build                                         # build the package
ssh prod "cd /srv/app && git pull && systemctl restart app"  # deploy
```

A task runner gives them short, memorable names and one place to read.

--

## `uv run` is itself a task runner

For many small projects, `uv run` is enough — no extra tool needed. The commands above are already two-word handles.

Reach for the tools on the next slides when:

- Tasks compose (run lint, then tests, then build)
- You want help text and option parsing (`--verbose`, `--coverage`)
- Some tasks need shell glue, others Python logic
- The project is polyglot and Make is already there

---

## Make — the classic

Save this as a file named **`Makefile`** at the project root. Make is older than Python, ubiquitous on Unix, and every dev already knows it.

```makefile
# Makefile
install:
	uv sync

test:
	uv run pytest -xvs tests/

lint:
	uv run ruff check . && uv run mypy src/

format:
	uv run ruff format . && uv run ruff check --fix .
```

--

## Make — running it

From the project root:

```bash
make install
make test
make lint
make format
```

`make` (no argument) runs the **first target** in the file — by convention the most-used one. Many projects make that `help` or `test`.

```bash
make             # → runs the first target (here: install)
```

To see what targets exist without running anything:

```bash
grep -E '^[a-z]+:' Makefile
# install:
# test:
# lint:
# format:
```

--

## Make — gotchas

- **Tabs, not spaces.** Indentation must be a literal tab character.
- **Each line is its own shell** — `cd foo` and `make` on the next line won't see the directory change. Chain with `&&` or `;` on one line.
- **`.PHONY` matters** — it tells make these targets are not file names. Without it, a file called `test` will skip the recipe.
- **Variables work but feel C-ish:**

```makefile
SRC = src/ tests/

lint:
	uv run ruff check $(SRC)
	uv run mypy $(SRC)
```

---

## invoke — Python-native

When your project lives in Python, write tasks in Python. Save as `tasks.py` at the project root.

```python
# tasks.py
from invoke import task

@task
def install(c):
    c.run("uv sync")

@task
def test(c, verbose=False):
    flag = "-xvs" if verbose else "-x"
    c.run(f"uv run pytest {flag} tests/")

@task(pre=[install])           # run install first
def ci(c):
    c.run("uv run ruff check . && uv run mypy src/ && uv run pytest")
```

--

## invoke — running it

```bash
uv add --dev invoke

uv run invoke test --verbose   # → runs the test task with verbose=True
uv run inv test -v             # short alias

uv run invoke --list           # see all tasks
# Available tasks:
#   ci
#   install
#   test
```

Each `@task`-decorated function becomes a CLI subcommand; parameters become flags.

--

## invoke — options and contexts

```python
@task(help={"name": "Whose name to greet"})
def greet(c, name="world", loud=False):
    """Say hello."""
    msg = f"Hello, {name}!"
    if loud:
        msg = msg.upper()
    print(msg)
```

```bash
uv run inv greet --name=Alice  # Hello, Alice!
uv run inv greet --loud        # HELLO, WORLD!
uv run inv --help greet        # see the docstring + options
```

The `c` (context) parameter is invoke's run-shell handle — it carries cwd, env, sudo, etc.

---

## click — CLI as a library

Often the right answer is "this isn't a task runner — it's our actual CLI". `click` makes that easy.

```python
# src/my_app/cli.py
import click

@click.group()
def cli(): pass

@cli.command()
@click.argument("name")
@click.option("--loud/--quiet", default=False)
def greet(name, loud):
    """Greet someone."""
    msg = f"Hello, {name}!"
    click.echo(msg.upper() if loud else msg)

if __name__ == "__main__":
    cli()
```

--

## click — exposing as a real binary

```toml
# pyproject.toml
[project.scripts]
my-app = "my_app.cli:cli"
```

```bash
uv sync                                # → installs the project (editable)
uv run my-app greet Alice --loud       # HELLO, ALICE!
```

Outside the project, expose globally:

```bash
uv tool install .                      # → `my-app` on PATH everywhere
my-app greet Alice --loud              # HELLO, ALICE!
```

---

## typer — click + type hints

Same idea, but driven by your function signatures.

```python
# main.py
import typer

app = typer.Typer()

@app.command()
def greet(name: str, loud: bool = False):
    """Greet someone."""
    msg = f"Hello, {name}!"
    typer.echo(msg.upper() if loud else msg)

if __name__ == "__main__":
    app()
```

```bash
uv run python main.py greet Alice --loud  # HELLO, ALICE!
uv run python main.py greet --help        # auto-generated help
```

`typer` reads your annotations, builds the parser, and integrates with `rich` for nice output.

---

## Choosing one

| Tool | When |
| --- | --- |
| **Make** | Polyglot project, team already speaks Makefile, simple shell glue |
| **invoke** | All-Python project, want richer logic + Python ecosystem |
| **click** | The "task runner" *is* the project's actual CLI |
| **typer** | Like `click`, but you want type-hint-driven ergonomics |

For a small project, `make` is hard to beat. For anything that grows complex shell logic, move it into `invoke` or `click`.

---

## What's next

- **PyTest** — testing as a first-class workflow
- **Static analysis** — ruff, mypy, pre-commit
