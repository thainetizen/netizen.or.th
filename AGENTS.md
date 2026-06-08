# AGENTS.md — Working notes for AI agents on netizen.or.th

This file briefs AI coding agents (Claude Code, GitHub Copilot, Cursor, etc.) on the project's stack, conventions, and constraints. Humans: see `PLAN.md` for the rollout plan; this file is for agents picking up work.

## What this project is

The website of the **Thai Netizen Network** (เครือข่ายพลเมืองเน็ต), a Thai civil-society organisation working on digital rights, freedom of expression, privacy, and an open internet. The site is bilingual (Thai default, English secondary), static, and deployed via GitOps to Cloudflare Pages.

Trustability and accessibility matter more than novelty — this is an NGO website, not a tech demo.

## Stack at a glance

- **SSG:** Hugo extended (`>= 0.161`).
- **Theme:** `github.com/cncf/dot-org-hugo-theme` pulled via Hugo Modules. Theme code lives in the Hugo module cache, not in `themes/`.
- **Config:** split-config dir under `config/` (`_default/`, `development/`, `production/`). **No root `hugo.toml`.**
- **CSS:** Dart Sass via the theme. Requires a real `sass` binary on PATH.
- **JS build:** `npm install` runs at build to pull `sass-embedded`, `postcss`, `autoprefixer`. Hugo Pipes handles the rest.
- **CMS:** Sveltia CMS (Decap-compatible) served from `static/admin/`.
- **Search:** Pagefind, built as a post-step from `public/`.
- **Hosting:** Cloudflare Pages. `_headers` and `_redirects` files at repo root.
- **Language:** Thai at `/`, English at `/en/`. Default content directory is `content/th/`.

## Repo layout

```
.
├── config/                  # split Hugo config (do not create hugo.toml at root)
│   ├── _default/
│   │   ├── hugo.yaml        # core settings, module imports, security
│   │   ├── languages.yaml   # th + en, menus, accessibility strings
│   │   └── params.yaml      # theme params, social links, github edit URL
│   ├── development/hugo.yaml
│   └── production/hugo.yaml # baseURL, robotsTXT, build.writeStats
├── content/
│   ├── th/                  # Thai content (root URLs)
│   │   ├── _index.md
│   │   ├── about.md
│   │   ├── contact.md
│   │   ├── statements/
│   │   ├── docs/
│   │   └── currents/
│   └── en/                  # English content (mirrors th/)
├── layouts/                 # local overrides only — theme provides base layouts
├── assets/                  # SCSS overrides, JS, images processed by Hugo Pipes
├── static/                  # files copied as-is (incl. /admin for Sveltia)
├── go.mod / go.sum          # Hugo Modules deps
├── package.json             # npm build deps (sass-embedded, postcss, autoprefixer)
├── postcss.config.js
├── PLAN.md                  # human-readable rollout plan, current status
└── AGENTS.md                # this file
```

## Commands

```bash
# PATH (Go is at /usr/local/go/bin, not default PATH)
export PATH="/usr/local/go/bin:/usr/local/bin:$PATH"

# Dev server
hugo server --configDir=config --buildDrafts --buildFuture

# Production build
hugo --configDir=config --environment=production --gc --minify

# Post-build search index
npx pagefind --site public

# Theme update
hugo mod get -u github.com/cncf/dot-org-hugo-theme && hugo mod tidy

# Install npm deps (needed for sass-embedded etc.)
npm install
```

Note: `node_modules/.bin/hugo` exists because the theme pulls `hugo-extended` as an npm dep, and it is an older version than the brew install. Prefer the system Hugo — keep `node_modules/.bin` *after* `/usr/local/bin` in PATH, or remove `hugo-extended` from `package.json` when stable.

## Content authoring conventions

Front-matter for every page (YAML):

```yaml
---
title: "หัวเรื่อง"            # required
description: "..."             # required; used for <meta description>
date: 2026-05-24               # required for statements / currents
translationKey: "stmt-2026-01" # links th↔en versions (REQUIRED for any page that has a translation)
tags: ["privacy", "surveillance"]
draft: false
---
```

Rules:
- Always create both `th/` and `en/` versions of a top-level page (`about`, `contact`, section indexes). Use the same `translationKey` so the lang switcher can pair them.
- Section indexes (`_index.md`) are required for `statements/`, `docs/`, `currents/` in both languages.
- File slugs in `th/` and `en/` can match (`about.md`/`about.md`) or differ. The lang switcher pairs by `translationKey`, not by filename.
- Dates: ISO 8601 (`2026-05-24`). Site timezone is `Asia/Bangkok`.
- Image alt text is required. Decorative images are exceptional — use `alt=""` only when truly decorative and add a comment explaining why.

## A11y rules (non-negotiable)

Target: **WCAG 2.2 AA**.

- Semantic landmarks (`<header> <nav> <main> <article> <aside> <footer>`). Don't replace with divs.
- Skip-to-content link, visible on focus.
- Visible focus indicator with ≥3:1 contrast (SC 2.4.11).
- Color contrast: 4.5:1 body, 3:1 large text and UI components.
- Strict heading order h1→h2→h3 — no skips.
- Per-page `<html lang>` matches the page's actual language.
- Inline `<span lang="...">` whenever a passage in one page is in a different language.
- Keyboard reachable: nav, search, lang switcher, every interactive control. No keyboard traps.
- Honor `prefers-reduced-motion`.
- Self-host Noto Sans Thai (`static/fonts/`). **Never** use Google Fonts CDN.

Before declaring an a11y task done: run axe (or WAVE) and do a keyboard-only walk.

## Privacy rules (non-negotiable)

- No third-party scripts or fonts loaded from external CDNs.
- No HTML forms that post anywhere. Contact is `mailto:` + downloadable PGP key.
- No cookies. Analytics is Cloudflare Web Analytics (edge-side, no JS).
- Don't migrate WP comments from the archive — PII risk, low value.
- `_headers` must set CSP, HSTS, Referrer-Policy strict, Permissions-Policy deny.

## What NOT to do

- Don't add features the user didn't ask for. This includes adding tracking, newsletter signups, donation forms with PII fields, third-party social embeds, etc.
- Don't add a root `hugo.toml`. Edit `config/_default/hugo.yaml` instead.
- Don't vendor the theme into `themes/` — it's a Hugo Module.
- Don't add `brew tap …` commands without confirming with the user first.
- Don't commit `node_modules/`, `public/`, `resources/`, `.hugo_build.lock` (`.gitignore` should cover; verify).
- Don't introduce deprecated Hugo keys: use `label`/`locale` (not `languageName`/`languageCode`) — Hugo 0.158+.
- Don't add emojis to content or commit messages unless explicitly requested.
- Don't push to `origin` until the user explicitly says so. Local-first development is the current mode.

## Branch & commit conventions

- Working locally on `main` until the project is mature enough to push to `https://github.com/thainetizen/netizen.or.th` as one big initial commit. Do not push without explicit instruction.
- After the initial public commit, future content edits from non-tech editors will come in via Sveltia CMS as PRs (branch prefix `cms/`). Maintainers merge after review (PR-as-editorial-workflow until Sveltia v1.0 ships native editorial workflow, expected mid-2026).
- Commit messages: short imperative subject; body explains *why* if non-obvious. No AI signatures.

## Open issues to know about

See `PLAN.md` for current blockers. As of the last checkpoint:

- **Dart Sass not yet on PATH** — theme build fails. Resume by installing Dart Sass (npm `sass`, or local tarball — *not* via a new brew tap without asking).

## Where to look first

- Resume plan and current status: `PLAN.md`.
- Theme source (read-only, in module cache): `~/Library/Caches/hugo_cache/modules/filecache/modules/pkg/mod/github.com/cncf/dot-org-hugo-theme@v0.1.8/`.
- Theme exampleSite (was copied to root once; re-fetch if needed): `https://github.com/cncf/dot-org-hugo-theme/tree/main/exampleSite`.
