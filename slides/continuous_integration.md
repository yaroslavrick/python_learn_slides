---
title: Continuous integration
---

![CI loop with stages and a green status](/assets/images/topics/continuous-integration.svg)
<!-- .element: class="title-illustration" -->

# Continuous integration

Run tests, lint, and type-check on every push.

---

## What CI gives you

- Tests run **on a clean machine** — not just yours
- Every push is verified before code review even starts
- Multiple Python versions, OSes, dependency sets in parallel
- Auto-deploy from the main branch when checks pass
- A green badge as the contract: this commit passed

CI exists because "works on my machine" isn't a guarantee.

---

## The feedback loop

```
push → CI runs → red/green ← review → merge
        ↑             ↓
        └── fix ──────┘
```

The ideal: under five minutes from push to result. Optimize for that.

---

## GitHub Actions — your first workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { enable-cache: true }
      - run: uv sync --frozen
      - run: uv run pytest
```

`astral-sh/setup-uv` installs uv, the project's Python (from `.python-version`), and caches dependencies — all in one step.

Push, open the **Actions** tab, watch it run.

---

## Adding lint and type-check

Run them as separate steps so failures point at the right thing.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { enable-cache: true }
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy src/
      - run: uv run pytest --cov=src --cov-report=term-missing
```

When CI is red, the failed step name is the headline.

---

## Caching dependencies

`astral-sh/setup-uv` caches by your `uv.lock` hash automatically — that's the whole story.

```yaml
- uses: astral-sh/setup-uv@v3
  with: { enable-cache: true }
- run: uv sync --frozen
```

Cold runs ~30 s → cached runs ~3 s.

--

## Caching for pip-based projects

For projects that haven't moved to uv yet, `actions/setup-python` can cache pip's directory:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
    cache-dependency-path: pyproject.toml
- run: pip install -e ".[dev]"
```

Less effective than uv's lock-based cache (no integrity hashes, coarser key) — but better than nothing.

---

## Matrix — many Python versions

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        python: ['3.11', '3.12', '3.13']
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { enable-cache: true }
      - run: uv sync --frozen --python ${{ matrix.python }}
      - run: uv run pytest
```

Nine jobs, all parallel. `fail-fast: false` lets you see all failures, not just the first.

---

## Splitting jobs for clarity

Linters fail fast — keep them separate from tests, and gate tests behind lint passing.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { enable-cache: true }
      - run: uv sync --frozen
      - run: uv run ruff check . && uv run ruff format --check .
      - run: uv run mypy src/
```

Lint takes seconds. Failing here means nobody waits on the test job.

--

## Gate tests behind lint

```yaml
  test:
    runs-on: ubuntu-latest
    needs: lint                      # ← only run if lint passed
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with: { enable-cache: true }
      - run: uv sync --frozen
      - run: uv run pytest
```

`needs:` gates expensive jobs behind cheap ones. Tests don't run on lint-broken code.

---

## pre-commit in CI

The same hooks you run locally — re-run in CI so nothing slips through.

```yaml
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - uses: pre-commit/action@v3.0.1
```

If `pre-commit run --all-files` is green locally, this job is fast and quiet.

---

## Other CI platforms

The concepts transfer — each ecosystem has its own YAML, identical substance.

--

## GitLab CI

```yaml
# .gitlab-ci.yml
test:
  image: ghcr.io/astral-sh/uv:python3.12-bookworm
  script:
    - uv sync --frozen
    - uv run pytest
```

The official `astral-sh/uv` image ships uv pre-installed, so no setup step is needed.

--

## CircleCI

```yaml
# .circleci/config.yml
version: 2.1
jobs:
  test:
    docker: [{ image: ghcr.io/astral-sh/uv:python3.12-bookworm }]
    steps:
      - checkout
      - run: uv sync --frozen
      - run: uv run pytest
```

Pick the platform your team already uses — the substance is identical.

---

## What CI shouldn't do

- Run flaky tests on retry until green — fix the test
- Hide warnings — promote them to errors (`-W error`, `filterwarnings = ["error"]`)
- Mask network failures with `continue-on-error` — they tell you something
- Take 30 minutes — split, parallelize, cache

CI you trust is CI you fix. CI you don't trust is CI you ignore.

---

## What's next

- **Django** — the batteries-included web framework
- **FastAPI** — modern API-focused framework
