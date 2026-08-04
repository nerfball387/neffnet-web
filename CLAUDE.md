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

## Automations write into this docroot

Four active n8n workflows write **directly into this working tree** on Mercury. Because the
docroot is the working tree, **their output is live on neffio.com the moment it is written** —
no commit, no pull, no deploy step.

| Workflow | Writes | Bind mount |
|---|---|---|
| Neff.io:Excel Tips Weekly Digest (Fridays 17:00 PT) | `neff.io/excel-digest/` + appends `PMXcodex/tips.json` | `/data/excel-digest`, `/data/pmxcodex` |
| Neff.io:PMX Submit Tip | `PMXcodex/tips.json` | `/data/pmxcodex` |
| Neff.io:PMX Tip Patch | `PMXcodex/tips.json` | `/data/pmxcodex` |
| Neff.io:PMX Authors Commit | `PMXcodex/authors.json` | `/data/pmxcodex` |

Mounts and env are in `/etc/systemd/system/n8n.service`.

GitHub is a **backup**, not the publish path. The systemd timer
`neffnet-digest-backup.timer` (every 15 min, `/usr/local/bin/neffnet-digest-backup.sh`)
commits and pushes those directories. It stages only the ones that actually changed, and
labels the commit accordingly:

| Changed | Commit message |
|---|---|
| digest only | `Back up Excel Tips digest YYYY-MM-DD` |
| Codex only | `Back up PMX Codex YYYY-MM-DD` |
| both | `Back up Excel Tips digest and PMX Codex YYYY-MM-DD` |

**This was flipped on 2026-08-03.** Everything above previously used GitHub API nodes as its
filesystem — reading a file from GitHub, patching it, writing back — with Pages publishing the
result. When Pages was retired that left output committed to GitHub but not live anywhere until
someone pulled. Writing to the serving origin first and replicating to GitHub after is the
correct direction.

Consequences:
- The backup job stages **only** `neff.io/excel-digest/` and `PMXcodex/`, so unrelated
  uncommitted work in the docroot is never swept into its commits
- Its commits are made on Mercury, so unlike the old GitHub-API commits they are already in
  the local tree — but still `git pull` before editing, in case of web edits
- A workflow bug is live immediately rather than sitting in a repo. Git history is the
  rollback path

Steve also edits pages directly in GitHub's web editor (commits titled `Update index.html`)
and pulls them down to Mercury afterwards.

### ⚠️ n8n runs the *published* version, not the draft

n8n 2.x keeps two copies of every workflow. Editing the canvas — or patching
`workflow_entity.nodes` in the DB — changes only the **draft**. Production reads the
**published** snapshot in `workflow_history`, selected by `workflow_entity.activeVersionId`.

- "Test workflow" (manual) runs the **draft**
- Schedule and webhook triggers run the **published** version

This burned a full afternoon on 2026-08-03: the digest's manual runs looked correct while every
webhook-triggered follow-on batch ran a months-old snapshot. Worse, un-publishing and
re-publishing re-activated *the same old version* rather than promoting the draft.

Verify a publish actually took, rather than trusting the button:

```bash
sudo python3 -c "
import sqlite3
c=sqlite3.connect('file:/opt/n8n/data/database.sqlite?mode=ro',uri=True)
print(c.execute(\"select name,versionId,activeVersionId from workflow_entity where active=1\").fetchall())"
```

`versionId` (draft) and `activeVersionId` (published) must match.

### ⚠️ n8n restricts local file access by default

n8n 2.x ships `N8N_RESTRICT_FILE_ACCESS_TO` defaulting to `~/.n8n-files`; in 1.x it was
unrestricted. Any path outside the allowlist fails with the unhelpful **"Access to the file is
not allowed."** — not a permissions error, despite the wording. The unit sets:

```
N8N_RESTRICT_FILE_ACCESS_TO=/data/excel-digest;/data/sermons;/data/pmxcodex
```

**Adding a new write target means adding both a bind mount and an entry here.** The same
message is also thrown when any parent directory of the target is a symlink.

### Reading a file in a Code node

Use `getBinaryDataBuffer`, never a base64 decode:

```js
const html = (await this.helpers.getBinaryDataBuffer(0, 'data')).toString('utf-8');
```

Binary data mode is `filesystem`, so `$input.first().binary.data.data` is a **stub, not the
payload**. Decoding it as base64 yields ~13 bytes of garbage. On 2026-08-03 that silently
overwrote the live `excel-digest/index.html`; recovery was `git checkout`.

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
PMXcodex/                  PM Excelerated Codex (written by the PMX workflows)
  index.html               the app; served at neffio.com/PMXcodex/
  tips.json                appended by Submit Tip, Tip Patch, and the Friday digest
  authors.json             whole-file overwrite by Authors Commit
  .image-slots.state.json  dot-prefixed; tracked, do not let a tool skip it
excel/  tips/  pics/
```

`landing/` was added 2026-08-02 — it previously lived at `/srv/neffio/landing`, outside any
repo and with no backup.

### PM Excelerated Codex — `PMXcodex/`

Deployed 2026-08-03. Served at **`https://neffio.com/PMXcodex/`** — the path is
**case-sensitive**, `/pmxcodex` is a 404. Admin UI is the `#admin` fragment on that same page.

Three webhook workflows back the admin UI. They are reachable at `n8n.neffio.com`, which the
tunnel exposes un-gated specifically for webhooks — `automate.neffio.com` is the Access-gated
editor and will not work for these:

```
https://n8n.neffio.com/webhook/pmx-submit
https://n8n.neffio.com/webhook/pmx-authors
https://n8n.neffio.com/webhook/pmx-tip-patch
```

The URLs are stored per-browser in localStorage, so they must be set once on each device used
for admin. Each webhook's CORS `allowedOrigins` is `https://neffio.com`; a change of origin
means editing all three.

Auth is an `X-PMX-Key` header compared against `$env.PMX_ADMIN_KEY` in each workflow's auth
Code node. **No key literal exists in any workflow.** The value lives in
`/etc/n8n-pmx.env` (mode 600, root-only) and is passed through by `--env PMX_ADMIN_KEY`, so it
is not in the world-readable unit file. Reading it requires `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`,
also set in the unit — without it every call fails 401 with no useful diagnostic.

The original deployment guide
(`OneDrive\03.Projects\PMX Codex\2026-07-27-pmx-deployment-guide.md`) predates the migration
and is **stale in three ways**: it targets `neffnet.com`, it has the workflows commit through
the GitHub API, and it routes webhooks over Tailscale Serve — Tailscale is not installed on
Mercury. The converted, in-use versions are in that folder under `n8n\converted-mercury\`.

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
