# neffnet-web — Claude Code Context

Static content for Steve Neff's personal sites. No build step, no framework — hand-written
HTML/CSS committed directly. What is in the repo is what gets served.

---

## ⚠️ This repo publishes to TWO places, with different rules

Getting this wrong is the easiest mistake to make here.

| Target | Served by | How it updates |
|---|---|---|
| **https://neffnet.com** | GitHub Pages (branch `main`, path `/`) | **Automatic** on push |
| **https://neffio.com** | nginx on Mercury, docroot `/srv/neffnet-web` | **Manual** — someone must `git pull` on Mercury |

So a push updates neffnet.com within a minute and does **nothing** for neffio.com. And a
local edit on Mercury is live on neffio.com immediately — committed or not — while
neffnet.com knows nothing about it until you push.

The two domains show **different home pages from the same repo**:
- `neffnet.com/` → `index.html` at the repo root (a plain placeholder)
- `neffio.com/`  → `landing/index.html`, via an nginx alias (see routing below)

Pages is enabled and `CNAME` contains `neffnet.com` — do not delete that file or Pages
stops resolving the custom domain.

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
- Those commits land on GitHub and auto-publish to neffnet.com without human involvement
- **Mercury does not see them until someone pulls**, so neffio.com can silently lag by a week
- Always `git pull` before editing here, or you will be working from a stale tree and may
  conflict with the automation

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
location /            -> try_files $uri $uri/ =404          # everything else
location ~ /\.git     -> deny all; return 404
```

- `location = /` is an **exact** match, which is why the apex serves `landing/index.html`
  while the repo's root `index.html` sits unused on neffio.com (Pages still uses it).
- The landing page references its assets as `/_landing/...` — that prefix exists only on
  Mercury. On neffnet.com those paths 404, which is expected.
- **The docroot is a git working tree.** `location ~ /\.git` is what stops `.git/` being
  downloadable over the web. Do not remove it.

After changing the vhost: `sudo nginx -t && sudo systemctl reload nginx`.

---

## Layout

```
CNAME                      neffnet.com — required by GitHub Pages
index.html                 root placeholder; neffnet.com's home, unused by neffio.com
landing/                   neffio.com's actual home page + logo assets
  index.html
  Neffio_logo_*.png
neff.io/excel-digest/      weekly Excel Tips digests (written by the n8n workflow)
todoist/                   "Lister" Todoist client — index.html + support.js runtime
excel/  tips/  pics/  PMXcodex/
```

`landing/` was added 2026-08-02 — it previously lived at `/srv/neffio/landing`, outside any
repo and with no backup. Adding it to the repo also means it is published at
`neffnet.com/landing/`, which is harmless but was not the goal.

### ⚠️ `todoist/` — never restore the hardcoded API token

Added 2026-08-03 from `OneDrive\03.Projects\AutomateMyLife\TaskManager\Todoist Interface.dc.html`.
It is a generated `.dc.html` bundle: `index.html` needs `support.js` beside it, and neither
is hand-edited — re-export from the authoring tool and re-apply the change below.

The source file ships `const DEFAULT_TOKEN = '<live 40-char Todoist token>'` near line 628.
**That line must never be committed.** The repo is public and also served by GitHub Pages, so
committing it hands anyone full read/write on Steve's Todoist account. In this repo the line
reads `(typeof window !== 'undefined' && window.__TODOIST_TOKEN) || ''`.
Re-check it on every re-export; the authoring tool puts the literal token back.

The real token lives in **`todoist/token.js`, which is gitignored and exists only on
Mercury.** It sets `window.__TODOIST_TOKEN`, so `/todoist/` auto-connects on neffio.com. On
neffnet.com that file 404s and the app falls back to its "Add API token" banner, which stores
whatever you paste in `localStorage` only. If you ever rebuild Mercury, recreate `token.js`
by hand — nothing restores it from git, by design.

**Cloudflare Access does not make it safe to commit the token.** Access gates
`neffio.com/todoist` (app `Lister (Todoist)`, allow sjneff@gmail.com). It has no reach over
GitHub Pages: `neffnet.com/todoist/index.html` is served publicly with no login, from the
same commit. Two publishing targets, one perimeter — that asymmetry is the whole reason for
the split.

---

## Conventions

- Plain HTML/CSS, no build tooling. Don't introduce a bundler or framework for this.
- Keep pages self-contained; assets live beside the page or in `pics/`.
- Test on **both** targets when it matters — the aliases mean neffio.com and neffnet.com do
  not render identically.
- Don't commit anything secret. This repo is **public**, and half of it is also served by
  GitHub Pages.

## Related

- A stale Windows clone exists at `C:\Users\steve\OneDrive\03.Projects\neffnet-web` and is
  being retired. Do not push, pull, or merge from it — Mercury is the authority.
- Mercury runs three other services behind the same Cloudflare Tunnel (n8n, the Lifespring
  Service Director, and this site). Tunnel config: `/etc/cloudflared/config.yml`.
