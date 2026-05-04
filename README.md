# Python Course

Slide-deck-driven course covering Python, web development with Django and FastAPI, testing, tooling, infrastructure, and a few data/AI extras. Built with [Jekyll](https://jekyllrb.com/) and [Reveal.js](https://revealjs.com/).

**Live at**: <https://yaroslavrick.github.io/python_learn_slides/>

## What's inside

- **Python language** — basics, OOP, metaprogramming, refactoring, design patterns
- **Tooling & testing** — uv, packaging, task runners, pytest, ruff/mypy, CI
- **Web — Django** — models/URLs/views/templates, auth, DRF, GraphQL, Celery, BDD
- **Web — FastAPI & async** — FastAPI, WSGI/ASGI, asyncio
- **Infrastructure** — Git, Docker, Ansible, deployment
- **Data & AI extras** — NumPy/pandas, scikit-learn, LLM agents

## Local development

### 1. Install Ruby

The project's Ruby version is pinned in `.tool-versions` (currently **4.0.3**). The simplest way to get matching Ruby is with [`mise`](https://mise.jdx.dev/) — a polyglot version manager that reads `.tool-versions` automatically.

<details>
<summary><strong>Click to expand: install Ruby with <code>mise</code></strong></summary>

#### Install `mise`

```bash
# macOS / Linux
curl https://mise.run | sh

# OR via Homebrew on macOS
brew install mise
```

Activate it in your shell — pick the snippet for your shell:

```bash
# zsh
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc

# bash
echo 'eval "$(mise activate bash)"' >> ~/.bashrc

# fish
echo 'mise activate fish | source' >> ~/.config/fish/config.fish
```

Restart the shell after editing the rc file.

> Already on `asdf`? You can keep using it — `asdf install` reads the same `.tool-versions` file. Skip ahead to step 2.

#### Install the project's Ruby

From the repo root:

```bash
mise install      # reads .tool-versions, downloads Ruby 4.0.3
```

Verify:

```bash
ruby --version    # ruby 4.0.3 ...
which ruby        # ~/.local/share/mise/installs/ruby/4.0.3/bin/ruby
```

When you `cd` into this repo, `mise` (or `asdf`) auto-switches to Ruby 4.0.3. When you leave, it switches back to your global default.

</details>

### 2. Install gems

```bash
gem install bundler           # if not already installed
bundle install                # installs Jekyll + plugins from Gemfile.lock
```

### 3. Vendor Reveal.js (one-time)

The `vendor/` directory is gitignored, so Reveal.js isn't in the repo. Clone it locally:

```bash
git clone --depth 1 --branch 5.1.0 https://github.com/hakimel/reveal.js.git vendor/reveal.js
rm -rf vendor/reveal.js/.git
```

(In CI, the GitHub Pages workflow does this automatically — see `.github/workflows/pages.yml`.)

### 4. Serve

```bash
bundle exec jekyll serve --config _config.yml,_config_dev.yml --livereload
```

Open <http://localhost:4000> — the dev config keeps `baseurl` empty so links work locally.

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

## Deployment

Pushes to `main` trigger `.github/workflows/pages.yml`, which:

1. Clones Reveal.js into `vendor/reveal.js`
2. Builds the Jekyll site (`bundle exec jekyll build`, with `JEKYLL_ENV=production`)
3. Uploads the `_site/` output and deploys to GitHub Pages

Roughly two minutes from `git push` to live.

## Licensing

This site bundles [Reveal.js](https://github.com/hakimel/reveal.js) (MIT license) under `vendor/reveal.js/` (cloned at build time). See `NOTICES.md` for attribution.
