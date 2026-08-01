# Hosting guide — putting this live

The site is static, so hosting is just "serve these files". It is set up for **GitHub Pages**
deploying straight from the `main` branch — no build, no CI, no deploy keys.

**Current state: Pages is not enabled.** `https://riturajsinghrajput.github.io/bookends-dashboard/`
returns **404**. Everything below is what turns that into a 200.

---

## 1. Enable GitHub Pages (one time, ~2 minutes)

1. Go to **https://github.com/RITURAJSINGHRAJPUT/bookends-dashboard**
2. **Settings** → **Pages** (left sidebar)
3. Under **Build and deployment → Source**, choose **Deploy from a branch**
4. **Branch:** `main` · **Folder:** `/ (root)` → **Save**
5. Wait ~1 minute. The Pages panel shows the live URL with a green tick.

That's it. No workflow file is needed — the repo has no build step, and a Pages *build* action is
added automatically for branch deploys.

**Your URLs once it's live:**

| Page | URL |
|---|---|
| Brand dashboard | `https://riturajsinghrajput.github.io/bookends-dashboard/` |
| Strategic Cockpit | `https://riturajsinghrajput.github.io/bookends-dashboard/cockpit/` |
| P&L updater | `https://riturajsinghrajput.github.io/bookends-dashboard/cockpit/update.html` |

Verify from the terminal:
```bash
curl -o /dev/null -w "%{http_code}\n" https://riturajsinghrajput.github.io/bookends-dashboard/
curl -o /dev/null -w "%{http_code}\n" https://riturajsinghrajput.github.io/bookends-dashboard/cockpit/
```
Both should print `200`.

## 2. Deploying changes

There is no deploy step. **Push to `main` and Pages rebuilds within ~1 minute.**

```bash
git add -A
git commit -m "…"
git push origin main
```

Two things also write to `main` on their own:
- The **daily refresh** workflow commits an updated `index.html` each morning ([SETUP-AUTOMATION.md](SETUP-AUTOMATION.md)).
- The **P&L updater** commits `cockpit/data.json` via the GitHub API when someone clicks Publish.

So `git pull` before you start editing, or you'll hit conflicts with the bot.

## 3. Read this before you share the link

> **The passcode gate protects the rendered page. It does not protect the repository.**

This repo is **public**. Anyone who knows the repo URL can read the raw files without ever seeing
the passcode prompt — verified:
```
https://raw.githubusercontent.com/RITURAJSINGHRAJPUT/bookends-dashboard/main/cockpit/data.json  → 200
```
That file is the complete FY27 P&L — revenue, EBITDA, COGS, parent loans. `index.html` likewise
carries the follower/reach figures, the TickTick backlog and the content calendar.

The gate is a client-side SHA-256 check whose hash sits in the page source. It is a **deterrent
against casual access**, plus the `noindex` tag keeps all three pages out of Google. It is not
access control. Treat the URL as the real secret.

If that isn't good enough for the P&L, you have three options:

| Option | Cost | Trade-off |
|---|---|---|
| **Make the repo private** | GitHub **Pro** (~$4/mo) | Pages from a private repo needs a paid plan — on Free, Pages only serves public repos. Closes the raw-file hole; the site itself stays passcode-only. |
| **Move to a host with real auth** | Free tier exists | Cloudflare Pages + Cloudflare Access, or Netlify with password protection, gives server-side auth. Requires migrating hosting; the `update.html` publish flow still needs the repo. |
| **Accept it** | — | Fine if the numbers are "internal, not catastrophic". Just know the link is the only barrier. |

### Do this first: the old deployment is still live and completely open

`https://vasuvekariyadome-design.github.io/bookends-dashboard/` returns **200** and serves the full
pre-transfer dashboard — same data, snapshot `2026-06-26`. Verified against it:

- **No passcode gate.** The copy there predates the fix; the gate had been dropped from `index.html`.
- **No `noindex` tag.** It is crawlable, so it can turn up in Google results.
- Its `raw.githubusercontent.com/vasuvekariyadome-design/…/cockpit/data.json` is also **200**.

Locking down this repo achieves nothing while that one is up. Get the old repo's **Settings → Pages →
Source → None**, or delete the repo outright.

## 4. Wire up the automation

Two separate credentials, neither of which transfers with a repo. Both are currently missing.

### Daily Instagram refresh — `IG_ACCESS_TOKEN`
The dashboard's numbers are baked into `index.html` by a nightly GitHub Actions run. It has not
committed since 2026-06-26, which is why the header pill reads a red *"Stale · N days old"*.

Full setup and the ordered diagnostic: **[SETUP-AUTOMATION.md](SETUP-AUTOMATION.md)**. Short version:
Settings → Secrets and variables → Actions → add `IG_ACCESS_TOKEN`, make sure Actions is enabled
(transferred repos arrive with workflows off), then Actions → *Daily dashboard refresh* → **Run workflow**.

### Monthly P&L publish — a fine-grained PAT, per editor
Whoever updates the cockpit needs their own token, pasted into `update.html` Step 5 and kept in their
browser's localStorage. Never committed.

- GitHub → Settings → Developer settings → **Fine-grained tokens** → Generate new
- **Repository access:** Only select repositories → `bookends-dashboard`
- **Permissions → Repository permissions → Contents: Read and write** (nothing else)

These tokens expire; regenerate when they do. Steps 1–5 of the workflow: [cockpit/README.md](cockpit/README.md).

`update.html` derives the target repo from its own URL when served from `*.github.io`, so moving the
repo to another account can never again silently publish to the old one. Served locally it falls back
to the constants in its `GH` block — update those if the repo is ever **renamed**.

## 5. Custom domain (optional)

1. Settings → Pages → **Custom domain** → enter e.g. `dashboard.bookends.in` → Save
2. At your DNS provider, add a **CNAME** for that subdomain → `riturajsinghrajput.github.io`
3. Wait for the DNS check to pass, then tick **Enforce HTTPS**

GitHub writes a `CNAME` file into the repo root — leave it there. Note the site moves to the domain
**root path**, so the cockpit becomes `https://dashboard.bookends.in/cockpit/`. Every internal link
is relative (`./cockpit/`, `../`), so navigation keeps working; `update.html`'s publish target
auto-detection is `*.github.io`-only and will fall back to its hardcoded constants, which are correct.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Pages URL 404s after enabling | Give it 2–3 min for the first build. Then check Actions → *pages build and deployment* for a red run. |
| Site is live but shows an old version | Pages caches hard. Hard-reload (`Cmd-Shift-R`), and confirm your commit is actually on `main` on github.com. |
| Cockpit renders but numbers look stale | It fell back to its embedded seed because `data.json` failed to load. Check Network for `data.json` → 200, and that the file is valid JSON. |
| Passcode prompt appears twice | Shouldn't — all three pages share `bookends_unlock_v1`. If it happens, one page's `HASH` doesn't match the others. |
| Header pill is red "Stale · N days old" | The daily refresh isn't running → [SETUP-AUTOMATION.md](SETUP-AUTOMATION.md). |
| Publish says 404/403 | The token lacks **Contents: Read and write** on this repo, or it expired. |

## Related

- [RUN.md](RUN.md) — running and editing it locally
- [SETUP-AUTOMATION.md](SETUP-AUTOMATION.md) — the daily refresh
- [cockpit/README.md](cockpit/README.md) — the monthly P&L workflow
