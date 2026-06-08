# Thai Netizen Network — netizen.or.th rebuild plan

Resume document. Pick up from here when the session restarts.

## Project goal

Revive the Thai Netizen Network website (formerly WordPress at https://thainetizen.org/) as a static, GitOps-published, bilingual (Thai + English) site at https://netizen.or.th. Old domain redirects to new. No databases, no PII collection.

## Stack (locked)

| Layer | Choice | Why |
|-------|--------|-----|
| SSG | Hugo (extended) | Multilingual built-in, fast at archive scale, single binary, CF Pages native. |
| Theme | [`cncf/dot-org-hugo-theme`](https://github.com/cncf/dot-org-hugo-theme) v0.1.8 | Built for orgs, WCAG 2.1 AA target, semantic HTML, multilingual-ready, MIT. |
| Theme integration | Hugo Modules + theme's npm/postcss config copied to root | Version pin via go.mod, override layer in `layouts/` and `assets/`. |
| CMS | [Sveltia CMS](https://sveltiacms.app/) with GitHub OAuth via Cloudflare Worker | Web-only contribution. Editors visit `/admin/`, sign in with GitHub, commit via UI. |
| Default language | Thai at `/`, English at `/en/` | Primary audience is Thai. |
| Search | Pagefind (already integrated by theme) | Static, multilingual, zero-config. |
| Hosting | Cloudflare Pages | Static, free, fast, integrated DNS. |
| Analytics | Cloudflare Web Analytics (no cookies) | No PII. |
| Repo | `https://github.com/thainetizen/netizen.or.th` (public, empty until push) | Public for transparency. Develop locally; push as one chunk when mature. |

## Three stages of rollout

### Stage 1 — Fresh netizen.or.th
Build new site. Bilingual nav, statements/docs/currents/about/contact sections. A11y baseline. Sveltia CMS. CF Pages deploy.

### Stage 2 — thainetizen.org redirect shell
Separate CF Pages project on the old domain. `_redirects` maps the 5 core paths to corresponding pages on netizen.or.th. Catch-all 302 to new root.

### Stage 3 — Archive dig
Wayback CDX enumeration → snapshot fetch → WP `.entry-content` → Markdown → commit + slug map → regenerate thainetizen.org `_redirects`. Skip comments (PII).

## Current state

Working dir: `/Users/art/projects/netizen.or.th` (local-only, not yet pushed).

### Done

- **Toolchain**: Hugo 0.162.1 extended (brew), Go 1.26.3 at `/usr/local/go/bin/go`, Node v26, Dart Sass binary at `node_modules/sass-embedded-darwin-x64/dart-sass/sass` (from `npm install`), npm-global `sass` 1.100.0 at `/usr/local/bin/sass`.
- **Hugo scaffold + Module**: `hugo new site .`, `module github.com/thainetizen/netizen.or.th`, CNCF theme v0.1.8 pulled.
- **Split config** under `config/{_default,development,production}/`. No root `hugo.toml`.
  - `defaultContentLanguage: th`, `contentDir: content/th/`, `defaultContentLanguageInSubdir: false`.
  - `security.exec.allow` includes `sass`, `go`, `node`, `npx`, `postcss`.
  - Languages use Hugo 0.158+ keys (`label`, `locale`).
- **Content skeleton**: `content/{th,en}/{_index,about,contact}.md` + `{statements,docs,currents}/_index.md`. `translationKey` pairs versions.
- **A11y overrides** in `layouts/`:
  - `partials/language-selector.html` — uses `.Label`, links via `.AllTranslations`.
  - `partials/head/custom-head.html` — self-hosted Noto Sans Thai (Regular/Medium/Bold woff2 in `static/fonts/`), `hreflang` alternates.
  - `partials/head/custom-css.html` — pipes `assets/css/netizen.css` through Hugo for fingerprint + SRI.
  - `assets/css/netizen.css` — focus-visible ring (3px solid #0052cc), skip-link styling, keyboard-friendly dropdowns (`:focus-within`), `prefers-reduced-motion`, Thai `line-height: 1.8`, external-link arrow with `speak: never`.
- **Theme already provides**: skip-link (`<a class="skip-link" href="#content">`), Pagefind integration (`/pagefind/` files emitted on build), search route.
- **Sveltia CMS**: `static/admin/{index.html,config.yml}` — uses `multiple_root_folders` i18n (matches Hugo layout exactly). Collections: `statements`, `currents`, `docs`, `pages` (home/about/contact as known files). Thai admin UI (`locale: th`), Unicode slugs.
- **Headers / redirects / robots**:
  - `static/_headers` — HSTS, CSP, X-Frame-Options SAMEORIGIN, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy denies camera/mic/geo/payment/usb/interest-cohort, COOP/CORP, font/css long cache, `/admin/*` no-store + noindex.
  - `static/_redirects` — placeholder (deep redirects come in Stage 3 on the thainetizen.org project).
  - `static/robots.txt` — disallow `/admin/`, sitemap link.
- **Build verified**: `hugo --configDir=config --environment=production --gc --minify` produces TH 19 / EN 17 pages, `/admin/`, `/pagefind/`, robots.txt, all clean.

### Open warnings (low priority)

Two deprecation warnings emitted by the CNCF theme's templates (not our overrides):
- `.Site.Languages` → use page-level access (theme to fix upstream).
- `.Site.LanguageCode` → use `.Site.Language.Locale`.

These will break in a future Hugo. Either patch the theme via local override or pin Hugo version until theme updates.

## Remaining todos

| # | Task | Notes |
|---|------|-------|
| 11 | Register GitHub OAuth App | github.com/settings/developers. Homepage `https://netizen.or.th`. Callback `https://sveltia-oauth.netizen.or.th/callback`. Save Client ID + Secret for Worker. |
| 12 | Deploy Cloudflare Worker OAuth proxy | Use [`sveltia/sveltia-cms-auth`](https://github.com/sveltia/sveltia-cms-auth). Secrets: `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`. Route `oauth.netizen.or.th/*`. Update `base_url` in `static/admin/config.yml`. |

### Later (not yet tasked)

- Push to `github.com/thainetizen/netizen.or.th` (one big initial commit when ready).
- Create CF Pages project: build command `npm install && hugo --configDir=config --environment=production --gc --minify`, output `public`, env `HUGO_VERSION=0.162.1` / `NODE_VERSION=22`.
- Bind custom domain `netizen.or.th` in CF Pages.
- GitHub Action: `pa11y-ci` / axe on `public/` per PR; HTML proofer for broken links.
- Stage 2: build `thainetizen-redirects` (or repo + CF Pages project for the old domain).
- Stage 3: Wayback CDX scraper, WP-HTML → Markdown pipeline, image rescue, slug map.

## Build / dev commands

```bash
# Required PATH (Go and the bundled Dart Sass)
export PATH="$(pwd)/node_modules/sass-embedded-darwin-x64/dart-sass:/usr/local/go/bin:/usr/local/bin:$PATH"

# Dev server
hugo server --configDir=config --buildDrafts --buildFuture

# Production build
hugo --configDir=config --environment=production --gc --minify

# CMS local (Chromium): start dev server, open /admin/, "Work with Local Repository"
# CMS remote: needs tasks #11 + #12 done

# Update theme module
hugo mod get -u github.com/cncf/dot-org-hugo-theme
hugo mod tidy
```

## Accessibility checklist

- WCAG 2.2 AA target.
- Semantic landmarks via theme.
- Skip-to-content link, visible on focus ✓ (theme + our CSS).
- Visible focus indicator ≥3:1 contrast (SC 2.4.11) ✓ (netizen.css).
- Color contrast 4.5:1 text / 3:1 large text and UI — verify per page when adding content.
- Strict heading hierarchy — review each new page.
- Per-page `<html lang>` ✓ (verified `lang="th"` / `lang="en"`).
- Image alt text required field in CMS schema ✓ (statements/currents `hero.alt`).
- Keyboard: nav, lang switcher, search reachable — `:focus-within` ensures dropdowns open.
- `prefers-reduced-motion` ✓.
- Thai font self-hosted ✓ (Noto Sans Thai woff2 in `static/fonts/`).
- Run axe/WAVE on real content once homepage/statements are populated.

## Privacy / no-PII rules

- No third-party scripts in base layout ✓ (Pagefind self-hosted, fonts self-hosted).
- No Google Fonts call ✓.
- Contact = `mailto:` + PGP key (PGP key TBD).
- Analytics via Cloudflare edge (configure when domain bound).
- `Permissions-Policy` denies camera/mic/geo ✓.
- Comments not migrated from archive (PII risk).

## Open decisions deferred

- Worker subdomain name: `oauth.netizen.or.th` vs `sveltia-oauth.netizen.or.th` vs `*.workers.dev`. Current config placeholder: `https://sveltia-oauth.netizen.or.th`.
- PR-based editorial flow: configure branch protection on `main` so CMS commits land on `cms/*` branches, maintainer reviews. (Sveltia v1.0 will add native editorial workflow — expected mid-2026.)
- Archive `robots` policy — index or `noindex`.

## How Hugo and Sveltia relate

They are separate tools coupled by convention — neither knows about the other at runtime.

```
┌─────────────┐  writes Markdown to  ┌──────────────┐  reads Markdown  ┌──────────────┐
│ Sveltia CMS │ ────────────────────▶│  GitHub repo │ ───────────────▶ │     Hugo     │
│  (browser)  │                      │  content/**  │                  │ (build step) │
└─────────────┘                      └──────────────┘                  └──────────────┘
      ▲                                                                       │
      │ editor edits                                                          │ outputs
      │ via /admin/                                                           ▼
                                                                       ┌──────────────┐
                                                                       │ static HTML  │
                                                                       │  (CF Pages)  │
                                                                       └──────────────┘
```

- **Hugo** — build tool. Reads `content/**/*.md` + templates → emits static HTML. Knows nothing about Sveltia.
- **Sveltia** — browser edit UI. Reads/writes those same `.md` files via GitHub API. Knows nothing about Hugo templates.
- **Contract** — `static/admin/config.yml`. Tells Sveltia which folders hold content, what front-matter fields exist, and the i18n structure. Hugo reads the same files independently.
- **Risk** — if Hugo's expected front-matter schema and Sveltia's `config.yml` drift apart, content breaks at build time. Keep them in sync.

## References

- CNCF theme: https://github.com/cncf/dot-org-hugo-theme
- Sveltia CMS: https://sveltiacms.app/
- Sveltia OAuth helper: https://github.com/sveltia/sveltia-cms-auth
- Sveltia i18n structures: `multiple_root_folders` matches Hugo's `content/<locale>/<section>/`.
- Pagefind: https://pagefind.app/
- Wayback CDX API: https://github.com/internetarchive/wayback/tree/master/wayback-cdx-server
