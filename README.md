# Python Course

Slide-deck-driven course covering Python, web development with Django and FastAPI, testing, tooling, infrastructure, and a few data/AI extras. Built with [Jekyll](https://jekyllrb.com/) and [Reveal.js](https://revealjs.com/).

## What's inside

- **Python language** — basics, OOP, metaprogramming, refactoring, design patterns
- **Tooling & testing** — uv, packaging, task runners, pytest, ruff/mypy, CI
- **Web — Django** — models/URLs/views/templates, auth, DRF, GraphQL, Celery, BDD
- **Web — FastAPI & async** — FastAPI, WSGI/ASGI, asyncio
- **Infrastructure** — Git, Docker, Ansible, deployment
- **Data & AI extras** — NumPy/pandas, scikit-learn, LLM agents

## Local development

Requires Ruby 4.x and Bundler.

```bash
bundle install
bundle exec jekyll serve --config _config.yml,_config_dev.yml --livereload
```

Site is served at <http://localhost:4000>.

## Authoring slides

Slides live under `slides/` as Markdown files. Reveal.js separators:

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

The `layout: slide` is set automatically for everything under `slides/` via the `defaults` rule in `_config.yml`.

## Licensing

This site bundles [Reveal.js](https://github.com/hakimel/reveal.js) (MIT license) under `vendor/reveal.js/`. See `NOTICES.md` for attribution.
