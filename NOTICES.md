# Third-party notices

This site uses the following third-party software, fetched as Python dependencies via `uv` and bundled in the deployed output:

## mkslides

- Source: <https://github.com/MartenBE/mkslides>
- License: MIT
- Pulled in as a Python package; see `pyproject.toml` and `uv.lock`.

## Reveal.js

- Source: <https://github.com/hakimel/reveal.js>
- License: MIT
- Bundled by mkslides and copied into `_site/mkslides-assets/reveal-js/` at build time.

## highlight.js (themes)

- Source: <https://github.com/highlightjs/highlight.js>
- License: BSD-3-Clause
- Theme CSS bundled by mkslides and copied into `_site/mkslides-assets/highlight-js-themes/` at build time.
