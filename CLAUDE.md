# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout: two git repos

- **Outer repo** (`kouandco/test1`): only tracks the standalone FRED Treasury-yields chart: `us_treasury_yields.py`, its output `us_treasury_yields.html`, and `index.html` (a copy of it served by GitHub Pages).
- **`kouandco.github.io/`**: a separate, nested git repo (`kouandco/kouandco.github.io`, served at seankou.com). It is untracked by the outer repo, so commit and push it from inside that directory. Most of the real work happens here.
- **`eurex_data/`**: an untracked archive of raw Eurex daily trade files (`efpf/`, `efs/`), the launchd plist that runs the hourly trade update, and its `update.log`.
- **`.venv/`** at the outer root is the shared Python env for both repos (pandas, plotly, requests, numpy, jupyter).

There are no tests, linters or build tooling. "Building" means running a data script and/or `quarto render`.

## Commands

Outer chart:
```bash
.venv/bin/python us_treasury_yields.py      # writes us_treasury_yields.html
cp us_treasury_yields.html index.html       # publish copy
```

Site (run from `kouandco.github.io/`):
```bash
export QUARTO_PYTHON=../.venv/bin/python    # index.qmd executes Python via Jupyter
quarto render index.qmd                     # BTP–Bund page; fetches FRED at render time
../.venv/bin/python build_baskets.py        # -> baskets.json, then: quarto render ctd.qmd
../.venv/bin/python build_eurex_flow.py     # -> eurex_flow.json, then: quarto render eurex-flow.qmd
../.venv/bin/python update_eurex_trades.py  # -> eurex_trades.json (no re-render needed)
```
Rendered `.html` files and `*_files/` dirs are committed (GitHub Pages serves them directly; `.nojekyll`). There is no `_quarto.yml`, so each page carries its own front matter (theme `[darkly, theme.scss]`).

## Site architecture

Each page is a pipeline: **Python builder → JSON in the site dir → Quarto page that loads it client-side with OJS**.

| Page | Builder | Data file | How the page loads it |
|------|---------|-----------|-----------------------|
| `index.qmd` | none (Python cells in the page) | none | Python executed at render time |
| `ctd.qmd` | `build_baskets.py` | `baskets.json` | `FileAttachment` |
| `eurex-flow.qmd` | `build_eurex_flow.py`, `update_eurex_trades.py` | `eurex_flow.json`, `eurex_trades.json` (+ `baskets.json` for CTD info) | `FileAttachment`, plus `fetch(...?v=Date.now())` for trades |

- `index.qmd` is the only page that runs Python. `ctd.qmd` and `eurex-flow.qmd` are pure `{ojs}` and do all their computation in the browser.
- `ctd.js` holds the single-factor Monte Carlo CTD model (yields in decimal, prices per 100, CFs per 1 face). `ctd.qmd` imports it dynamically with a cache-busting query (`ctd.js?v=${Date.now()}`) so a cached old copy is never mixed with a new page. Keep that pattern for runtime-loaded modules and data that changes without a re-render.
- `charts.js` holds the shared Plotly dark-theme layout and color palette (`COLORS`, `layout()`). Use it for new charts so they stay visually consistent.

### Data-source quirks that drive the design
- **Eurex only keeps short windows online**: about 20 trading days of trade files and the latest 5 monthly statistics files. `update_eurex_trades.py` therefore archives every trade file into `../eurex_data/` and re-downloads the last two days, because the current day's file grows during the session. `build_eurex_flow.py` appends each month to `eurex_flow_history.csv`. Never delete these archives, because the history can't be re-fetched.
- **Eurex trade file scaling**: futures prices have 3 implied decimals for FGBS/FGBM/FBTS and 2 for the other contracts. Cash prices have 4 and nominal is in thousands. Swap fixed rates are usually 4 implied decimals, but some rows use 2 with a decimal point. See the `update_eurex_trades.py` docstring.
- **Contract rolls are manual**: `build_baskets.py` hard-codes the front contracts (`UST_MONTH`, `UST_LAST_DAY`, `EUREX_DELIVERY`, and per-contract delivery dates in `UST_CONTRACTS`). Roll these forward at each quarterly expiry.
- All scrapers send the same browser `User-Agent` (the `UA` constant in each script).
- Eurex product codes map to Bloomberg-style tickers throughout: FGBS→DU, FGBM→OE, FGBL→RX, FGBX→UB, FOAT→OAT, FBTS→BTS, FBTP→IK.

### Scheduled job
`eurex_data/com.seankou.eurex-trades.plist` runs `update_eurex_trades.py` hourly via launchd. It uses absolute paths under `/Users/seankou/Documents/project`, so update the plist if the project moves. Output goes to `eurex_data/update.log`.
