# Session 5 Prompt — Price Charts + Good-Buy Engine

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -30` — expected to end with Session 0–4 commits (scaffold, data layer, scraper + Newegg, scheduler, dashboard UI). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — green before changes; start the app and confirm `/` renders with your current data.

## Where we are
Sessions 0–4 are done: prices auto-flow in the background for active lists (Newegg items); a working dashboard exists with list sections, toggles, add-item box, item rows with sparklines, and an **empty** "Good Buy Opportunities" section placeholder. No per-item chart page and no deal scoring exist yet. Your job is both, per **PLAN.md §7–§8**.

## Your job this session (deliverables)

### 1. Vendored Chart.js (no CDN — the app must work on a LAN with zero external calls)
- Download the Chart.js UMD bundle into `app/static/js/chart.min.js` and commit it. Note the version in a comment at the top of the file or in SESSION_LOG.md.

### 2. History API
`GET /api/items/{id}/history?range=7d|30d|90d|all` → JSON:
```json
{"points": [{"t": "ISO-8601", "price_cents": 12345, "in_stock": true}],
 "stats": {"min": ..., "max": ...}}
```
Only prices within the selected range; oldest→newest order.

### 3. Good-buy engine — `app/buys.py` (pure logic over price rows, no DB coupling beyond a fetch helper)
Implement **PLAN.md §7** exactly:
- Rolling stats per item from its `prices` history: last-90-day median / min / mean; all-time tracked low.
- `insufficient_data` flag when the item has < 7 days of price history (not ranked, shown as "insufficient data" — never faked analytics).
- **good_buy** badge when current price ≤ `(1 − GOOD_BUY_PCT/100) ×` 90-day median, OR at/below the 30-day minimum with a drop ≥ `DROP_ALERT_PCT`% vs the previous scrape.
- **hot_deal** badge when current price is the all-time tracked low (or ≤ 90-day low minus a small margin).
- `deal_score`: primary = % below 90-day median; tie-break by recency of the drop; in-stock items rank above out-of-stock.
All thresholds come from config (`GOOD_BUY_PCT`, `DROP_ALERT_PCT`) — no magic numbers outside this module.

### 4. Dashboard integration
- Replace the empty-state placeholder: query all active-list items, compute buys via the engine, sort by `deal_score` desc, and render the top section as deal cards showing title/site/price + badges (e.g. "↓18% vs 90-day avg", "All-time low", "Dropped $42 since last check").
- Items with insufficient data are excluded from ranking but may appear in a clearly-labeled sub-note; the empty state remains for when there is truly nothing to show.

### 5. Item detail page — `GET /items/{id}` (`app/templates/item.html`)
- Full Chart.js **price-over-time line chart** fed by the history endpoint (range selector buttons: 7d / 30d / 90d / all that re-fetch and redraw).
- Stats block for the selected range (min/max/avg, points count).
- Recent price table (last ~30 rows: timestamp, price, in-stock).
- **"Scrape now"** button → `POST /api/items/{id}/scrape` which runs the shared `scrape_item` service once and returns the updated item JSON; page refreshes state after it.

### 6. Tests
- **Engine unit tests with synthetic series** (deterministic): price dropping below the 90-day median threshold → good_buy; all-time low → hot_deal; <7 days of data → insufficient_data and unranked; out-of-stock tie-break behavior. Build small in-memory price lists, no DB or network needed.
- History API tests (fixture-backed item with known rows); page render test for `/items/{id}` (200 + chart script present).

## Hard rules (apply to this and every session)
1. **Scope:** only this prompt's deliverables. No new site adapters (S6), no seed-data population (S7). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **No network in tests** — synthetic price data and fixtures only. Never insert fake price rows into the real dev DB to "prove" a badge; use unit tests for logic, and live verification only against genuinely scraped prices (see done-criteria experiment below).
3. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
4. **Commits:** one logical change per commit; imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
6. **Blockers:** stop if stuck; record under "Open issues" with what you tried; leave the tree coherent — never end with unexplained half-work.
7. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] `/items/{id}` renders a real Chart.js line chart for an item that has genuinely scraped history; range buttons switch data correctly.
- [ ] Good-buy section ranks items per §7 using only real data. **Controlled experiment** to prove ranking end-to-end without fake data: run the dev app with `GOOD_BUY_PCT=99` (any price ≤ 100% of its median qualifies), confirm the top section populates and ordering is sensible, then restore the default and re-verify the empty/insufficient state. Document this in SESSION_LOG.md.
- [ ] "Scrape now" on the item page produces a fresh price point (visible as a new row/point).
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Vendor Chart.js locally for offline charting" — Type: chore
2. "Add price history API with range filter" — Type: feature
3. "Implement good-buy scoring engine per plan §7" — Type: feature
4. "Render ranked Good Buy Opportunities on dashboard" — Type: feature
5. "Add item detail page with chart, stats, scrape-now" — Type: feature
6. "Cover buys engine and history API with tests" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 5** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 5 — <YYYY-MM-DD>
   - What was done: <bullets, incl. the GOOD_BUY_PCT=99 experiment result>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets>
   - Notes for next session: <what S6 adapters must conform to, and any buys-engine hooks they should know about>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, live chart/deal evidence (which item), and confirmation that **Session 6** is ready (`prompts/session-6.md`).
