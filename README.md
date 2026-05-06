# Python Course

Slide-deck-driven course covering Python, web development with Django and FastAPI, testing, tooling, infrastructure, and a few data/AI extras. Built with [mkslides](https://github.com/MartenBE/mkslides) (Python) and [Reveal.js](https://revealjs.com/).

**Live at**: <https://yaroslavrick.github.io/python_learn_slides/>

## What's inside

- **Python language** — basics, OOP, metaprogramming, refactoring, design patterns
- **Tooling & testing** — uv, packaging, task runners, pytest, ruff/mypy, CI
- **Web — Django** — models/URLs/views/templates, auth, DRF, GraphQL, Celery, BDD
- **Web — FastAPI & async** — FastAPI, WSGI/ASGI, asyncio
- **Infrastructure** — Git, Docker, Ansible, deployment
- **Data & AI extras** — NumPy/pandas, scikit-learn, LLM agents

## Local development

### 1. Install `uv`

The build runs on Python via [uv](https://docs.astral.sh/uv/). The Python version is pinned in `.python-version` (currently **3.14**).

<details>
<summary><strong>Click to expand: install <code>uv</code> and Python</strong></summary>

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# OR via Homebrew on macOS
brew install uv
```

`uv` reads `.python-version` and downloads matching Python on first sync.

</details>

### 2. Install dependencies

```bash
uv sync
```

This creates `.venv/` and installs `mkslides` (and its bundled Reveal.js) from `uv.lock`.

### 3. Serve locally

```bash
uv run mkslides serve slides/
```

Opens `http://localhost:8000` with live reload as you edit `.md` files under `slides/`.

To produce a static build instead:

```bash
uv run mkslides build slides/ -d _site/
```

## Authoring slides

Slides live under `slides/` as `.md` files. Reveal.js separators:

- `---` on its own line — new horizontal slide
- `--` on its own line — new vertical slide (nested under the current horizontal)
- `Note:` — speaker notes (not shown to audience)

Each slide file needs front matter:

```markdown
---
title: My Topic
---

# My Topic

---

## Section 1

content
```

The custom `templates/slide.html.jinja` and `templates/index.html.jinja` define the page chrome and landing-page layout; mkslides renders every `.md` deck through the slide template, plus `index.html` from the index template.

## Deployment

Pushes to `main` trigger `.github/workflows/pages.yml`, which:

1. Sets up `uv` and installs Python + dependencies from `uv.lock`
2. Runs `uv run mkslides build slides/ -d _site/`
3. Uploads `_site/` and deploys to GitHub Pages

Roughly a minute from `git push` to live.

## Licensing

This site uses [Reveal.js](https://github.com/hakimel/reveal.js) (MIT license) bundled by [mkslides](https://github.com/MartenBE/mkslides) (MIT). See `NOTICES.md` for attribution.
