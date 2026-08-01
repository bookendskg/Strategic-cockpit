# Daily refresh procedure — Bookends dashboard

This repo serves a **static** dashboard at
`https://riturajsinghrajput.github.io/bookends-dashboard/`.
The page has no live data connection when opened on the web, so fresh numbers must be
**baked into `index.html`**.

> **There are two refresh paths. This document describes the second one.**
>
> 1. **Automated (normal).** `.github/workflows/refresh.yml` runs `scripts/refresh.mjs`
>    every morning on GitHub's servers, hitting the **Meta Graph API** directly. Nothing to
>    do by hand. Setup and troubleshooting: [SETUP-AUTOMATION.md](SETUP-AUTOMATION.md).
> 2. **Agent-driven (manual fallback, below).** A daily agent holding the **Supermetrics +
>    TickTick** MCP tools does the same job interactively. Use this when the workflow is
>    broken, or when you also want the TickTick backlog count refreshed — `refresh.mjs`
>    does not touch it.
>
> Both write the same `index.html`; they differ only in data source. Don't run both in the
> same morning.

The agent's job each morning: pull fresh Instagram metrics, rewrite the data inside
`index.html`, commit, and push. GitHub Pages redeploys automatically (~1 min).

Currency/locale: INR, India. **Never invent numbers** — if a query fails for a brand,
leave that brand's data untouched and note it in the commit message.

## Brands (Supermetrics ds_id = `IGI`, Instagram Insights)

| Brand    | account id            | history start |
|----------|-----------------------|---------------|
| Capiche  | `17841418368091658`   | last 90 days  |
| Aiko     | `17841449386057518`   | last 90 days  |
| Bookends | `17841463855235063`   | since 2024-06-20 |

## Step 1 — pull posts for each brand

```
data_query(
  ds_id="IGI",
  ds_accounts="<account id>",
  fields="timestamp,media_type,media_like_count,media_comments_count,media_saved,media_shares,media_reach,interactions,media_permalink,caption",
  date_range_type="custom",
  start_date=<90 days ago, or 2024-06-20 for Bookends>,
  end_date=<today>,
  max_rows=300, compress=true
)
```
Then poll `get_async_query_results(schedule_id=...)` until `status=="completed"`.
The data_query call sometimes times out before returning the schedule_id — just retry it.

⚠️ **CRITICAL — column order is NOT the order you requested.** Supermetrics returns **all
dimensions first, then all metrics**. Do NOT assume `r[2]` is likes. Map columns by the
**row-0 header names** (or `requested_field_ids`). For the field list above, the verified
returned order is:

```
index: 0          1           2               3        4      5        6      7       8      9
field: timestamp  media_type  media_permalink caption  likes  comments saves  shares  reach  interactions
```

(Dimensions = timestamp, media_type, media_permalink, caption — returned first in that
order; then the metrics in requested order.) Always re-check the header row in case the set
changes; never hard-trust these indices blindly.

## Step 2 — build each post row

Target row shape (this is what `index.html` expects):
```
[date, type, likes, comments, saves, shares, reach, interactions, url, caption]
```
- `date`  → `timestamp` sliced to `YYYY-MM-DD`
- `type`  → map `VIDEO`/`REELS` → `"Reel"`, `CAROUSEL_ALBUM` → `"Carousel"`, `IMAGE` → `"Image"`
- numbers → integers (`0` if blank)
- `url`   → `media_permalink`
- `caption` → **preserve curation:** if this post's `url` already exists in index.html, KEEP
  its current caption (these are hand-written editorial summaries — do not overwrite them).
  Only for a genuinely NEW url, generate one from the real caption: take the first line,
  trim to ~40 chars at a word boundary, strip `"` and newlines so it is safe inside a JS
  string. If empty, use `""`.
Keep the existing row order (the file is sorted by interactions/likes descending); just
update the numbers in place and insert any new posts. The UI re-sorts anyway.

## Step 3 — rewrite `index.html`

For each brand, replace the contents of that brand's `posts:[ ... ]` array (inside the
`const DATA={...}` block) with the freshly built rows. Replace **only** the rows — keep the
surrounding `name/handle/acct/color/followers/media/thr/deep` fields exactly as they are.

Also update the snapshot date:
```
const SNAPSHOT_DATE="<today YYYY-MM-DD>";
```

Leave the `deep:{...}`, `EXTRA`, `BENCH`, `CAL`, `STORYP` blocks unchanged — they are
labelled "snapshot" in the UI on purpose. (Followers/media counts also change slowly; leave
them unless explicitly asked to refresh.)

## Step 4 — TickTick backlog (optional, if TickTick MCP is available)

`get_project_with_undone_tasks(project_id="6a2cfbb9ebcdba000000029f")`, count the undone
tasks, and update `const BACKLOG={... total:<n> ...}`. If TickTick is unavailable, skip it.

## Step 5 — commit and push

```
git add index.html
git commit -m "Daily refresh — <today>  (Capiche/Aiko/Bookends posts)"
git push origin main
```
If a brand's query failed, say so in the commit message and do not touch that brand's rows.

## Verify

After pushing, the live page updates within ~1 minute. Confirm `SNAPSHOT_DATE` advanced to
today, and that the header pill reads **"Updated · <today>"** in amber.

The pill computes its own age from `SNAPSHOT_DATE`: up to 7 days it reads "Updated · <date>";
past that it switches to **"Stale · N days old"**, turning red past 21 days. If you see a
red pill, the refresh has stopped running — start at [SETUP-AUTOMATION.md](SETUP-AUTOMATION.md).
