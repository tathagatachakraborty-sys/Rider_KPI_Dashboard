# Last-Mile Ops Console

A single-page dashboard for the **Rider Weekly / Store Weekly / City Weekly** KPIs
that your notebook writes into the "Rider KPI Dashboard Data" Google Sheet.
It reads the sheet live (as CSV) every time the page loads — no backend, no API key,
just a static page you can host on GitHub Pages.

## ⚠️ First: rotate your Databricks token

`rider_kpi__2_.ipynb` has a live Databricks personal access token hardcoded in plain
text in the second cell. That notebook should **never** be committed to GitHub as-is.
Revoke/rotate that token in Databricks (User Settings → Access tokens) and, if you do
want the notebook in version control, load the token from an environment variable
instead, e.g. `access_token=os.environ["DATABRICKS_TOKEN"]`.

This repo/site only needs the **Google Sheet**, not the notebook, so you can keep the
notebook out of the repo entirely.

## 1. Make the Google Sheet readable by the page

The page fetches each tab as CSV via Google's `gviz` export endpoint, which only
works if the sheet is shared publicly (view-only):

1. Open **Rider KPI Dashboard Data** in Google Sheets.
2. **Share → General access → Anyone with the link → Viewer.**
3. Copy the Sheet ID out of the URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART_IS_THE_ID`**`/edit`

> This makes the KPI numbers viewable by anyone with the link — it does not expose
> your Databricks credentials or raw PII columns from `Merged Data`, since that tab
> isn't used by the page. Don't publish a sheet that contains anything sensitive
> beyond the three summary tabs.

## 2. Plug the Sheet ID into the page

Open `index.html`, find this block near the top of the `<script>`:

```js
const CONFIG = {
  SHEET_ID: "PASTE_YOUR_GOOGLE_SHEET_ID_HERE",
  TABS: {
    city:  "City Weekly",
    store: "Store Weekly",
    rider: "Rider Weekly"
  }
};
```

Replace `PASTE_YOUR_GOOGLE_SHEET_ID_HERE` with the ID from step 1. If you ever rename
the worksheet tabs, update the `TABS` values to match exactly.

## 3. Put it on GitHub Pages

```bash
# from an empty local folder
git init
git add index.html README.md
git commit -m "Rider/Store/City KPI console"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

Then in the GitHub repo: **Settings → Pages → Build and deployment → Source: Deploy
from a branch → Branch: `main` / `(root)` → Save.**

GitHub will give you a URL like `https://<your-username>.github.io/<repo-name>/`.
It can take a minute or two to go live after the first push.

## 4. Keeping it current

Every time someone opens the page, it re-fetches the CSVs, so it's always as fresh as
the last time your notebook ran `set_with_dataframe(...)` and updated the sheet. If
you schedule the notebook (cron, Airflow, Databricks job, etc.) to run weekly, the
dashboard updates itself with zero extra work.

## What's on the page

- **Level tabs** — switch between City / Store / Rider weekly summaries.
- **Filters** — week, city, store (rider tab also filters by store), plus a text
  search for rider ID / store code / city.
- **KPI cards** — totals and averages for the current filter selection.
- **Trend chart** — orders (bars) vs. OTD% (line) across weeks.
- **Top-N chart** — highest-volume cities/stores/riders in the current selection.
- **Sortable, paginated table** — click any column header to sort; OTD is shown as a
  color-coded badge (green ≥95%, amber ≥85%, red below).

`Merged Data` (the 55k-row order-level tab) isn't pulled in — it's too heavy for a
browser table. If you want a drill-down into raw orders later, that's a reasonable
follow-up (e.g. a paginated fetch, or pre-aggregating a 4th "daily" tab).
