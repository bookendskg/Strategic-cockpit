# Run guide — working on this locally

There is **nothing to install**. No `package.json`, no build step, no dependencies to fetch.
Every page is a single self-contained HTML file with its CSS and JS inlined; the only external
things are Chart.js, the XLSX/PDF parsers and the web fonts, all loaded from CDNs at runtime.

You do, however, have to **serve it over HTTP**. Double-clicking the file does not work — see
[Why not file://](#why-not-file) below.

---

## Start the server

Any static server works. Pick one:

### Option A — Python (already on macOS, nothing to install)
```bash
cd /Users/rituraj/Downloads/KG/bookends-dashboard
python3 -m http.server 8000
```
Then open **http://localhost:8000/**

### Option B — VS Code Live Server
The repo ships a port setting at [.vscode/settings.json](.vscode/settings.json) (`5501`).
Install the *Live Server* extension → right-click `index.html` → **Open with Live Server**.
Opens **http://127.0.0.1:5501/index.html**

### Option C — Node, if you prefer
```bash
npx serve .          # or: npx http-server -p 8000
```

Stop the server with `Ctrl-C`. To kill a stray one:
```bash
pkill -f "http.server 8000"
```

## The pages

| URL | File | What it is |
|---|---|---|
| `/` | [index.html](index.html) | Brand dashboard — This Week, Overview, per-brand tabs, Backlog, Content Planner |
| `/cockpit/` | [cockpit/index.html](cockpit/index.html) | Strategic Cockpit — 9 FY27 tabs, reads `data.json` |
| `/cockpit/update.html` | [cockpit/update.html](cockpit/update.html) | Monthly P&L updater — parses a Tally export, publishes `data.json` |

You can move between them in the UI: **◈ Strategic Cockpit** in the dashboard header,
**← Brand Dashboard** in the cockpit header, **← Back to cockpit** in the updater.

## The passcode

All three pages are behind the same SHA-256 gate. Enter the **team passcode** once and it covers
all three — they share one localStorage key (`bookends_unlock_v1`), and the two older per-page keys
are still accepted so nobody gets re-prompted.

To clear the unlock and see the gate again (useful when testing it):
```js
localStorage.clear()   // in DevTools console, then reload
```

To **change** the passcode: compute the SHA-256 of the new phrase and replace the `HASH` constant
near the top of **all three** files — `index.html`, `cockpit/index.html`, `cockpit/update.html`.
```bash
printf '%s' 'your new passcode' | shasum -a 256
```

## Why not file://

Opening the HTML directly (`file:///…/index.html`) breaks two things:

1. **The passcode gate.** It uses `crypto.subtle.digest`, which browsers only expose in a
   [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts).
   `https://` and `localhost` / `127.0.0.1` qualify; `file://` does not. The gate catches this
   and shows *"Open this via the live https link to unlock."*
2. **The cockpit's data load.** `fetch("./data.json")` is blocked by CORS on `file://`. The cockpit
   falls back to its embedded seed copy, so it still renders — but you are looking at stale baked-in
   numbers, not `data.json`. Confirm which you got: if `#loaderr` is empty, the fetch succeeded.

`localhost` is a secure context, so **plain HTTP on localhost is fine** — you don't need HTTPS locally.

## Editing

Everything is inline, so just edit the HTML and reload. No watcher, no rebuild.

Two things to be careful about:

- **Keep LF line endings.** [.gitattributes](.gitattributes) pins `index.html`, `cockpit/data.json`
  and `*.mjs` to LF because [scripts/refresh.mjs](scripts/refresh.mjs) rewrites `index.html` with
  regexes that match on `\n`. CRLF silently breaks the daily refresh.
- **Don't reshape the `DATA` block by hand** in `index.html`. `refresh.mjs` anchors on the exact
  shapes `acct:"…"`, `posts:[`, a two-space-indented `],` and `deep:{`. Changing that formatting
  makes the automated refresh a no-op. Check the anchors still match after any structural edit:
  ```bash
  grep -c 'posts:\[' index.html      # expect 3
  grep -c 'acct:"' index.html        # expect 3
  ```
- **`cockpit/data.json` is the cockpit's single source of truth** — every number lives there, in
  **₹ Lakhs** (`266.3` = ₹2.66 Cr). Validate after hand-editing:
  ```bash
  node -e 'JSON.parse(require("fs").readFileSync("cockpit/data.json","utf8"));console.log("valid")'
  ```

## Testing the publish flow locally

`cockpit/update.html` writes `cockpit/data.json` straight to GitHub via the Contents API — there is
no local write path. When served from `localhost` it targets the constants in the `GH` block
(`RITURAJSINGHRAJPUT/bookends-dashboard`); when served from `*.github.io` it derives the target from
the page URL instead. So **publishing from localhost commits to the real repo.** If you only want to
check the parser and the maths, use **Download data.json** instead of **Publish**.

## Smoke test

After any change worth checking, walk this in the browser:

1. `localStorage.clear()`, reload `/` → gate appears, body doesn't scroll behind it, a wrong
   passcode shows an error.
2. Unlock → dashboard renders, header pill shows freshness, all 7 tabs switch.
3. **Backlog** tab → the one item with a due date shows its ⏰ badge in amber.
4. **◈ Strategic Cockpit** → loads with **no second passcode prompt**, all 9 tabs present, and
   `#loaderr` is empty (meaning it read `data.json`, not the embedded seed).
5. **← Brand Dashboard** → back, still unlocked.

Check the DevTools console is clean and the Network tab shows `data.json` as **200**.

## Related

- [HOSTING.md](HOSTING.md) — putting it live on GitHub Pages
- [SETUP-AUTOMATION.md](SETUP-AUTOMATION.md) — the daily Instagram refresh, and what to do when it stops
- [REFRESH.md](REFRESH.md) — the manual/agent refresh fallback
- [cockpit/README.md](cockpit/README.md) — the cockpit's monthly P&L workflow
