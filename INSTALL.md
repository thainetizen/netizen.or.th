# INSTALL — deploy netizen.or.th on the free web

Step-by-step path from local repo to a public live site, using only free or free-tier services.

**Free services used:**
- GitHub (public repo, unlimited)
- Cloudflare Pages (500 builds/month, unlimited bandwidth, 100 custom domains)
- Cloudflare Workers (100k requests/day — way more than CMS sign-ins need)
- Cloudflare DNS (unlimited)
- Cloudflare Web Analytics (cookie-free, unlimited)
- Let's Encrypt TLS (auto via Cloudflare)

**Time to first deploy:** ~45 min if you already own `netizen.or.th`. Most of it is waiting for DNS.

---

## 0. Prereqs (one-time, local machine)

Skip any item already installed.

```bash
# Hugo extended
brew install hugo

# Go (required for Hugo Modules)
# Already at /usr/local/go/bin/go on this Mac. Otherwise:
#   brew install go

# Node 22+ (CF Pages build env uses 22)
node --version   # should be >= 22

# Dart Sass — the bundled native binary in node_modules will be used.
# No separate install required.
```

Verify the local build works:

```bash
cd /path/to/netizen.or.th
npm install
export PATH="$(pwd)/node_modules/sass-embedded-darwin-x64/dart-sass:/usr/local/go/bin:/usr/local/bin:$PATH"
hugo --configDir=config --environment=production --gc --minify
```

You should see `Total in <2s` and `public/` populated. If not, fix locally before touching the web.

---

## 1. Push the code to GitHub

The repo `github.com/thainetizen/netizen.or.th` already exists and is empty.

```bash
cd /path/to/netizen.or.th

# Verify .gitignore excludes build artifacts
grep -E '^(public|resources|node_modules|\.hugo_build\.lock)' .gitignore

# Stage everything except ignored files
git add -A
git status   # eyeball — confirm no node_modules / public

# First public commit
git commit -m "Initial public commit: bilingual Hugo site, Sveltia CMS, CNCF theme"

# Add remote and push
git remote add origin https://github.com/thainetizen/netizen.or.th.git
git branch -M main
git push -u origin main
```

Verify: open `https://github.com/thainetizen/netizen.or.th` — files visible.

---

## 2. Add the domain to Cloudflare (DNS)

You need `netizen.or.th` in a Cloudflare account so Pages can serve it on the apex and so the Worker subdomain works.

1. Sign up / sign in at https://dash.cloudflare.com (free plan).
2. **Add a site** → enter `netizen.or.th` → pick the **Free** plan.
3. Cloudflare scans existing DNS. Review records, then click **Continue**.
4. Cloudflare gives you two nameservers, e.g. `dana.ns.cloudflare.com` / `nicholas.ns.cloudflare.com`.
5. At your registrar (THNIC for `.or.th`), replace the current nameservers with the two from Cloudflare. Save.
6. Wait for propagation — usually < 1 hour for `.or.th`. Check `whois netizen.or.th` or `dig NS netizen.or.th`. Status in Cloudflare dashboard turns **Active** when ready.

While waiting, continue with steps 3–5 — they don't need DNS active yet.

---

## 3. Create the Cloudflare Pages project

1. In Cloudflare dashboard → **Workers & Pages** → **Create application** → **Pages** tab → **Connect to Git**.
2. Authorise the Cloudflare GitHub App, grant access to `thainetizen/netizen.or.th`.
3. Pick the repo. Settings:

   | Field | Value |
   |-------|-------|
   | Project name | `netizen-or-th` |
   | Production branch | `main` |
   | Framework preset | **Hugo** |
   | Build command | `npm install && hugo --configDir=config --environment=production --gc --minify` |
   | Build output dir | `public` |
   | Root dir | (leave blank) |

4. **Environment variables** (Production scope):

   | Name | Value |
   |------|-------|
   | `HUGO_VERSION` | `0.162.1` |
   | `NODE_VERSION` | `22` |

5. Click **Save and deploy**.

Watch the build log. First build takes ~2 min. When green, the site is at `https://netizen-or-th.pages.dev`. Visit it — both `/` (Thai) and `/en/` (English) should render.

---

## 4. Bind `netizen.or.th` and `www.netizen.or.th` as custom domains

DNS must be **Active** in Cloudflare for this step.

1. In the Pages project → **Custom domains** → **Set up a custom domain**.
2. Enter `netizen.or.th` → continue. Cloudflare adds the CNAME automatically since the DNS zone is in the same account.
3. Repeat for `www.netizen.or.th`.
4. Wait ~1 min for TLS cert issuance (Let's Encrypt, auto via Cloudflare).
5. Test in browser: `https://netizen.or.th` loads, `https://www.netizen.or.th` redirects to apex (or vice-versa — set the preferred one).

Optional: in the Pages domain settings, redirect `www` → apex (or apex → `www`, your call). Cloudflare also offers always-HTTPS — leave on.

---

## 5. Register the GitHub OAuth App (for Sveltia CMS web sign-in)

1. Go to https://github.com/settings/developers → **OAuth Apps** → **New OAuth App**.
2. Fill in:

   | Field | Value |
   |-------|-------|
   | Application name | `Thai Netizen CMS` |
   | Homepage URL | `https://netizen.or.th` |
   | Authorization callback URL | `https://sveltia-oauth.netizen.or.th/callback` |

3. Click **Register application**.
4. **Copy the Client ID** — paste it somewhere safe.
5. Click **Generate a new client secret** → **copy immediately** (shown once).

Keep both values handy for step 6.

---

## 6. Deploy the OAuth proxy as a Cloudflare Worker

The Sveltia auth helper handles the GitHub OAuth dance so editors never see a token.

### 6a. Deploy the Worker

```bash
# Anywhere outside the netizen.or.th repo (keep auth proxy separate)
cd /tmp
git clone https://github.com/sveltia/sveltia-cms-auth
cd sveltia-cms-auth

# Install Wrangler if needed
npm install -g wrangler

# Login (browser pops up)
wrangler login

# Deploy
wrangler deploy
```

The deploy prints a `*.workers.dev` URL — note it for now.

### 6b. Add the OAuth secrets

```bash
wrangler secret put GITHUB_CLIENT_ID
# paste the Client ID from step 5, hit enter

wrangler secret put GITHUB_CLIENT_SECRET
# paste the Client Secret, hit enter
```

Optional but recommended — restrict who can use this proxy:

```bash
wrangler secret put ALLOWED_DOMAINS
# paste: netizen.or.th
```

### 6c. Bind a clean subdomain

1. Cloudflare dashboard → **Workers & Pages** → click the deployed Worker (`sveltia-cms-auth`).
2. **Settings** → **Domains & Routes** → **Add Custom Domain**.
3. Enter `sveltia-oauth.netizen.or.th` → confirm.
4. Cloudflare creates the DNS record and binds the route.
5. Wait ~30s for TLS, then visit `https://sveltia-oauth.netizen.or.th/auth` — should show a small JSON or an error about missing query params. Either is fine — proves it's live.

### 6d. Verify the callback URL matches

Open the GitHub OAuth App settings (step 5) and confirm callback is `https://sveltia-oauth.netizen.or.th/callback`. If you used the `*.workers.dev` URL during step 5, edit it now.

The `static/admin/config.yml` already points at `https://sveltia-oauth.netizen.or.th` — no code change needed. If you chose a different subdomain, edit that file and push:

```bash
# Only if subdomain differs
git commit -am "cms: update OAuth base_url"
git push
```

---

## 7. Test web-only content editing

1. Open `https://netizen.or.th/admin/` in any browser.
2. Click **Sign in with GitHub**. Approve the OAuth scope.
3. You land in Sveltia. Click **Statements** → **New** → fill TH + EN tabs → **Save**.
4. Sveltia commits to `main`. Cloudflare Pages auto-builds (~90s).
5. Refresh `https://netizen.or.th/statements/` — your post appears.

If sign-in fails:
- Check the GitHub OAuth App callback exactly matches the Worker URL.
- Check Worker logs: `wrangler tail` (from the auth repo dir).
- Check `ALLOWED_DOMAINS` if set — must include `netizen.or.th`.

---

## 8. Enable Cloudflare Web Analytics (free, no cookies)

1. Cloudflare dashboard → **Analytics & Logs** → **Web Analytics** → **Add a site**.
2. Choose **Use existing zone** → `netizen.or.th`.
3. Done. No JS snippet needed — uses edge-side analytics. Zero impact on users, no GDPR cookie banner required.

---

## 9. Lock down `main` (so CMS commits go through PRs)

Optional but recommended once you have collaborators.

1. GitHub repo → **Settings** → **Branches** → **Add rule** for `main`.
2. Enable:
   - Require a pull request before merging
   - Require approval from at least one reviewer
3. Update `static/admin/config.yml` so Sveltia uses a working branch — add under `backend:`:

   ```yaml
   backend:
     name: github
     repo: thainetizen/netizen.or.th
     branch: main
     # Open PRs against main instead of committing directly
     open_authoring: false
     cms_label_prefix: "cms/"
   ```

   (Editorial workflow proper lands in Sveltia v1.0 — until then this is a soft convention.)

Commit, push. Reviewers see PRs from the CMS instead of direct pushes.

---

## 10. (Stage 2) thainetizen.org redirect shell

When ready to redirect the old domain:

1. Add `thainetizen.org` to the same Cloudflare account (repeat step 2 for that domain).
2. Create a separate tiny repo `thainetizen-redirects` with one file:

   ```
   # _redirects
   /about       https://netizen.or.th/about       301
   /statements  https://netizen.or.th/statements  301
   /docs        https://netizen.or.th/docs        301
   /currents    https://netizen.or.th/currents    301
   /contact     https://netizen.or.th/contact     301
   /*           https://netizen.or.th/:splat      302
   ```

3. Create a second CF Pages project pointing at this repo. Build command empty, output dir `.` (or a folder that contains `_redirects`).
4. Bind `thainetizen.org` and `www.thainetizen.org` as custom domains.
5. Done — old links forward.

Deep-link redirects (Stage 3) replace the catch-all once the Wayback archive is digested.

---

## Free-tier headroom

| Service | Free limit | Expected usage |
|---------|-----------|----------------|
| GitHub public repo | Unlimited storage | < 100 MB even after archive |
| CF Pages builds | 500/month | < 50/month for typical NGO cadence |
| CF Pages bandwidth | Unlimited | — |
| CF Workers | 100k requests/day | Auth: ~10/day; well within |
| CF Worker CPU | 10ms/request × 100k/day | Auth proxy is single-digit ms |
| CF DNS queries | Unlimited | — |
| CF Web Analytics | Unlimited | — |
| Let's Encrypt | Free | — |

Nothing here will charge you. The only paid path you'd hit is buying the domain itself (`.or.th` from THNIC, separate cost) and that's out of scope of this file.

---

## Rollback / disaster recovery

- **Bad deploy:** CF Pages keeps every prior build. Dashboard → Deployments → click an older green build → **Rollback to this deployment**. Instant.
- **Lost OAuth secrets:** redeploy the Worker, `wrangler secret put` fresh values, regenerate GitHub OAuth client secret. ~5 min.
- **Lost repo locally:** `git clone https://github.com/thainetizen/netizen.or.th` — the repo is the canonical state.
- **Lost domain:** outside our control (THNIC). Keep registration in good standing; turn on auto-renew if THNIC supports it.

---

## Resume points if interrupted

After step 1: site exists on GitHub only.
After step 3: site is live at `*.pages.dev`.
After step 4: site is live at `netizen.or.th`.
After step 6: CMS works at `/admin/`.
After step 8: analytics flowing.

Each step is independently safe to leave mid-way. None require sequencing beyond what the deps make obvious.
