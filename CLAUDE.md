# neffnet-web — Claude Code Context

Static content for Steve Neff's personal sites. No build step, no framework — hand-written
HTML/CSS committed directly. What is in the repo is what gets served.

---

## One repo, one site (since 2026-08-03)

This repo is served **only** by nginx on Mercury at **https://neffio.com**, docroot
`/srv/neffnet-web`. Deploys are manual — a push does nothing until someone pulls here.

GitHub is still the remote, and that part is load-bearing: it is the offsite copy and the
target the n8n digest workflow writes to (see below). Only the *hosting* was removed.

**GitHub Pages was disabled and neffnet.com retired on 2026-08-03.** Previously this same
repo was also published by Pages at neffnet.com, which is what let `/todoist/` be world-
readable while `neffio.com/todoist` was gated by Cloudflare Access — one repo, two origins,
two different security perimeters. Retiring Pages removed that whole class of bug.
`CNAME` was deleted with it; do not re-add it.

Consequence to remember: **nothing about this repo is public-facing except through Mercury.**
If Mercury is down, the site is down — there is no second origin anymore.

### Deploying to neffio.com

```bash
cd /srv/neffnet-web && git pull
```

No restart needed; nginx serves from disk. Verify with:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://neffio.com/
```

---

## An automation commits to this repo

The n8n workflow **"Neff.io:Excel Tips Weekly Digest"** (active, runs weekly on Mercury)
writes to this repo through the **GitHub API** — it uses GitHub nodes, not a local git
checkout. Commits appear as `Add/Update Excel Tips digest weekly_MM.DD.YYYY`.

Consequences:
- Those commits land on GitHub only. Since Pages was retired **nothing publishes them** —
  the digest is not live on neffio.com until someone pulls on Mercury
- So neffio.com can silently lag the archive by a week or more. This got *more* important
  when Pages went away: Pages used to publish these automatically, and now nothing does
- Always `git pull` before editing here, or you will be working from a stale tree and may
  conflict with the automation
- The workflow emails links to `https://neffio.com/excel-digest/...`, so a digest it just
  committed 404s for you until that pull happens

Steve also edits pages directly in GitHub's web editor (commits titled `Update index.html`)
and pulls them down to Mercury afterwards.

---

## nginx routing on Mercury

Vhost: `/etc/nginx/sites-available/neffio` (symlinked into `sites-enabled/`).
Docroot is this repo. Notable rules:

```
root /srv/neffnet-web;

location = /          -> alias /srv/neffnet-web/landing/   # EXACT match: the apex only
location /_landing/   -> alias /srv/neffnet-web/landing/   # landing page's own assets
location /excel-digest/ -> alias /srv/neffnet-web/neff.io/excel-digest/
location /todoist/    -> alias /srv/todoist/                # OUTSIDE the repo, see below
location /            -> try_files $uri $uri/ =404          # everything else
location ~ /\.git     -> deny all; return 404
```

- `location = /` is an **exact** match, which is why the apex serves `landing/index.html`
  while the repo's root `index.html` sits unused.
- The landing page references its assets as `/_landing/...`, an nginx-only prefix.
- **The docroot is a git working tree.** `location ~ /\.git` is what stops `.git/` being
  downloadable over the web. Do not remove it.

After changing the vhost: `sudo nginx -t && sudo systemctl reload nginx`.

---

## Layout

```
index.html                 root placeholder; unused (was neffnet.com's home before Pages was retired)
landing/                   neffio.com's actual home page + logo assets
  index.html
  Neffio_logo_*.png
neff.io/excel-digest/      weekly Excel Tips digests (written by the n8n workflow)
excel/  tips/  pics/  PMXcodex/
```

`landing/` was added 2026-08-02 — it previously lived at `/srv/neffio/landing`, outside any
repo and with no backup.

### ⚠️ `/todoist/` is served from OUTSIDE this repo — do not re-add it

The "Lister" Todoist client is at `neffio.com/todoist/` but its files live in
**`/srv/todoist/`**, not here. `todoist/` is gitignored so it cannot be re-added by accident.

It was briefly committed here on 2026-08-03 (`eeb360a`, `73b5afa`) and removed the same day,
because at the time **this repo was also republished by GitHub Pages at `neffnet.com`, where
Cloudflare Access did not apply.** Access gates `neffio.com/todoist` (app
`Lister (Todoist)`, allow sjneff@gmail.com) but had no reach over Pages, so the committed
copy was world-readable at `neffnet.com/todoist/`. Pages has since been retired, but the app
stays outside the repo — Access is enforced per-path at Cloudflare, and keeping the files out
of a public repo is a second, independent layer.

Nothing sensitive is in this repo's history — the API token was never committed (verified by
scanning every commit), which is why no history rewrite was needed.

Layout in `/srv/todoist/`, all three files required together:

```
index.html    generated .dc.html bundle; not hand-edited
support.js    its runtime; must sit beside index.html
token.js      sets window.__TODOIST_TOKEN — the live Todoist API token
```

Re-exporting from the authoring tool
(`OneDrive\03.Projects\AutomateMyLife\TaskManager\Todoist Interface.dc.html`) puts a literal
`const DEFAULT_TOKEN = '<live 40-char token>'` back around line 628. Replace it with
`(typeof window !== 'undefined' && window.__TODOIST_TOKEN) || ''` and copy to `/srv/todoist/`
— never into this repo. Without `token.js` the app falls back to its "Add API token" banner,
which stores what you paste in `localStorage` only.

`/srv/todoist/` is outside every backup and outside git, by design. If Mercury is rebuilt,
recreate `token.js` by hand; it is one line.

---

## Conventions

- Plain HTML/CSS, no build tooling. Don't introduce a bundler or framework for this.
- Keep pages self-contained; assets live beside the page or in `pics/`.
- Don't commit anything secret. This repo is **public on GitHub** even though it is no
  longer published by Pages.

## Related

- A stale Windows clone exists at `C:\Users\steve\OneDrive\03.Projects\neffnet-web` and is
  being retired. Do not push, pull, or merge from it — Mercury is the authority.
- Mercury runs three other services behind the same Cloudflare Tunnel (n8n, the Lifespring
  Service Director, and this site). Tunnel config: `/etc/cloudflared/config.yml`.
