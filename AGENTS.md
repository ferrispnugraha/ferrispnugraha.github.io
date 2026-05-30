# AGENTS.md — ferrispnugraha.github.io

## What this is

A personal academic portfolio site for Ferris Prima Nugraha (PhD candidate, HKUST ECE). Built with **Jekyll** + the **al-folio** theme. Deployed to GitHub Pages.

## Stack (Ruby, not Node)

- **Jekyll** static site generator, **Bundler** for deps, **SCSS** for styles
- **No package.json, no npm, no TypeScript, no JS/TS linter, no test framework**
- JS libs loaded via CDN in layouts (jQuery, Bootstrap, MathJax, Masonry, etc.)

## Development commands

```bash
# Install deps (run once)
bundle install

# Dev server with live reload
bundle exec jekyll serve --watch --port=8080 --host=0.0.0.0 --livereload --verbose --trace

# Production build
JEKYLL_ENV=production bundle exec jekyll build

# CI build (uses LSI)
bundle exec jekyll build --lsi
```

## Docker workflow

```bash
# Use prebuilt image (fast)
docker-compose up

# Build image from source then run
docker-compose -f docker-local.yml up
```

The Docker CMD deletes `Gemfile.lock` before starting (`rm -f Gemfile.lock`). That file is gitignored anyway.

## CI/CD

- **deploy.yml** — pushes to main trigger a GitHub Actions build (Ruby 3.2.2) that runs `bundle exec jekyll build` and deploys `_site/` to GitHub Pages via `JamesIves/github-pages-deploy-action`
- **deploy-image.yml** — guarded by `if: github.repository_owner == 'alshedivat'` (won't run on this fork)
- CI also installs `jupyter` (pip) and `mermaid.cli` (npm) at build time — some posts use notebooks/diagrams

## Repo quirks / gotchas

- **`future: true` in `_config.yml`** — blog posts with future dates are published. All current posts are dated Aug 2025 (drafts/pre-scheduled). New posts will show immediately.
- **`Gemfile.lock` is gitignored** — always regenerated locally/CI
- **`_site/` is gitignored** — generated output, do not edit directly
- **`assets/` is in `.dockerignore`** (excluded from Docker build but tracked in git — mounted as volume at runtime)
- **Giscus config** in `_config.yml` still points to `alshedivat/al-folio` (upstream placeholder) — update repo/repo-id if enabling comments
- **`_data/repositories.yml`** still has upstream placeholder entries (torvalds, twbs/bootstrap, etc.)
- **`.pre-commit-config.yaml`** enables basic hooks (trailing-whitespace, end-of-file-fixer, check-yaml, check-added-large-files) — run `pre-commit install` to activate

## Key paths

| Path | Purpose |
|------|---------|
| `_config.yml` | Site config (name, socials, theme settings) |
| `_layouts/` | HTML templates (default.html is base entrypoint) |
| `_pages/about.md` | Homepage content (permalink: /) |
| `_pages/publications.md` | Publications (powered by jekyll-scholar + `_bibliography/papers.bib`) |
| `_posts/` | Blog posts (Markdown + YAML front matter) |
| `_projects/` | Project entries |
| `_news/` | News announcements |
| `_bibliography/papers.bib` | BibTeX publication DB |
| `_plugins/` | Custom Jekyll plugins (cache-bust, details, external-posts, file-exists, hideCustomBibtex) |

## Adding content

- **Posts**: add `.md` files to `_posts/` with `layout: post` front matter
- **Projects**: add entries to `_projects/` (Markdown with YAML front matter including `img` and `categories`)
- **News**: add entries to `_news/` (Markdown with `date` and `inline` front matter)
- **Publications**: edit `_bibliography/papers.bib` and rebuild
