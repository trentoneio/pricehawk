# PriceHawk — Local Tech Price Tracker (Project Plan)

- **Status:** PLANNING only. No code exists yet; build happens in the sessions below.
- **Location:** `C:\Users\Trenton\coding-project\pricehawk`
- **Target host:** TrueNAS SCALE, deployed as a single Docker container managed through Dockage.
- **Scope (v1):** tech products — Amazon, Micro Center, Newegg, Best Buy, Server Parts Deals (forum).

---

## 1. What we're building

A single-user, self-hosted web app that tracks tech product prices over time across multiple retailers:

- **Add items by pasting a product URL.** Site is auto-detected; title/SKU are extracted automatically.
- **Lists with tracked/untracked toggle.** Items belong to named lists ("High Demand Tech", "Home Project", …). The per-list switch stops/starts all scraping for that list and hides/dims it on the dashboard.
- **Price vs. time charts** per item (7d / 30d / 90d / all-time ranges).
- **Background scheduler** scrapes each tracked item on an interval (default every 6h, staggered, rate-limited), storing a price point per run.
- **"Good Buy Opportunities" first.** The dashboard leads with items whose current price is meaningfully below their recent average/low, ranked by deal strength — everything else comes after.
- **Deployment:** one container on TrueNAS via Dockage; SQLite database persisted in a bind volume so data survives rebuilds/upgrades.

## 2. Non-goals (v1)

- No multi-user auth (single trusted user on the LAN).
- No proxies, no push/email alerts (UI badges only), no mobile app.
- No price prediction; good-buy logic is purely historical-statistics based.

## 3. Architecture (one container, one process)

```
                 ┌──────────────────────────── Docker container: pricehawk ────────────────────────────┐
 Browser ─────►  │  FastAPI web app (port 8000)                                                         │
   LAN :8321     │    ├─ Jinja templates + vendored Chart.js      ← UI                                 │
                 │    ├─ REST endpoints (/api/items, /api/lists, /api/history, toggles, scrape-now)    │
                 │    └─ APScheduler (in-process)                                                       │
                 │          │ every SCRAPE_INTERVAL_HOURS, staggered                                   │
                 │          ▼                                                                           │
                 │  SiteAdapters: Newegg | MicroCenter | BestBuy | Amazon | ServerPartsDeals           │
                 │          │ httpx client (UA rotation, delay+jitter, retries/backoff)                │
                 │          ▼                                                                           │
                 │  Retailer websites                                                                   │
                 └───────────────┬──────────────────────────────────────────────────────────────────────┘
                                 ▼
                     SQLite DB at /app/data/pricewatch.db   (bind volume ./data on the NAS)
```

- One container keeps Dockage deployment trivial: one service, one port, one volume.
- Scheduler runs in-process (APScheduler). No separate worker container needed at this scale (~dozens of items).

## 4. Tech stack & rationale

| Layer | Choice | Why |
|---|---|---|
| Language/runtime | Python 3.12 (`python:3.12-slim` image) | Best scraping ecosystem, simple single-container deploy |
| Web/API | FastAPI + Jinja2 templates | Fast to build; server-rendered pages keep the UI stack small |
| Charts | Chart.js (vendored into `static/`, no CDN dependency) | Works fully offline on a LAN NAS |
| DB | SQLite via SQLAlchemy 2.x, WAL mode | One file, trivially backed up/persisted, fast enough for this data volume |
| Scheduling | APScheduler (in-process) | Simple interval jobs with per-item staggering |
| HTTP/scraping | httpx + BeautifulSoup4 (+ JSON-LD helper); Playwright only as a last-resort adapter option | Keeps image small; headless browser added only if a site demands it |
| Tests | pytest, fixture HTML files for each adapter | Site-scrape regressions caught offline without hammering live sites |

## 5. Data model

```sql
lists        (id PK, name UNIQUE, active BOOL default true, created_at)
items        (id PK, list_id FK→lists, url UNIQUE, site TEXT, title TEXT, sku NULL,
              added_at, last_price_cents INT NULL, last_scraped_at, status TEXT 'ok'|'error', error_msg TEXT)
prices       (id PK, item_id FK→items, price_cents INT, in_stock BOOL, source_url, scraped_at)   -- the chart data; index (item_id, scraped_at)
scrape_runs  (id PK, item_id FK, started_at, finished_at NULL, outcome TEXT 'ok'|'price_changed'|'error', detail TEXT)
```

- `prices` is append-only time series — everything in §7 and the charts derives from it.
- `scrape_runs` gives observability: per-item success/failure history, surfaced on the logs page.

## 6. Retailer strategy (adapter pattern)

All sites implement one interface: `parse_url(url) → {site,title,sku}` and `fetch_price(item) → PricePoint(price_cents,in_stock,url)`. Failures are isolated per item and logged to `scrape_runs`; a bad adapter never crashes the scheduler.

| Site | Approach (try in order) | Difficulty | Notes |
|---|---|---|---|
| Newegg | 1) embedded page JSON / JSON-LD, 2) DOM parse fallback | Medium | Good first target — usually clean structured data on product pages |
| Micro Center | Static Magento-style HTML + JSON-LD | Low–Medium | Also has in-store stock info; nice "in stock" signal later |
| Best Buy | 1) headers + JSON-LD, 2) headless Playwright adapter if bot-walled | Hard | Akamai protection expected; budget extra time here |
| Amazon | Best-effort headers/JSON-LD only; **ToS prohibits scraping** — treat as optional | Very hard (legal + technical) | Recommended path later: third-party price API (e.g., Keepa) behind the same adapter interface |
| Server Parts Deals | Parse forum deal threads (title/product + price from post body) | Low–Medium | It's a deals board, not a catalog — supplemental source; lower priority |

**Shared client layer (all adapters):** rotating User-Agent pool, per-site minimum delay + jitter (default ≥2s), timeouts, 3 retries with exponential backoff, per-item circuit breaker (after N consecutive failures: pause that item's interval ×2 up to a cap, mark `status='error'` with the last error message).

## 7. Good-buy logic

Computed on read from the `prices` table (cheap in SQLite):

- Rolling stats per item: median/min/mean over last 90d; all-time low (tracked history).
- **Good buy** badge when current price ≤ (1 − GOOD_BUY_PCT) × 90-day median, *or* at/below the 30-day minimum with a drop ≥ DROP_ALERT_PCT vs. previous scrape.
- **Hot deal** badge when current price = all-time tracked low or at/below the 90-day low minus margin.
- `deal_score` ranks the dashboard: primarily % below 90-day median, tie-break by recency of the drop. In-stock items rank above out-of-stock ones.
- Cold start: items with <7 days of history show "insufficient data" and are not ranked — no faked analytics.

## 8. UI spec

| Page | Contents |
|---|---|
| `/` Dashboard | **Good Buy Opportunities** section first (cards: big current price, % vs 90-day avg, all-time-low badge). Below: one section per list with its name + tracked/untracked toggle switch; each item row = title, site, last price, mini sparkline, status icon. Add-item box at top: paste URL → submit auto-detects site/title (adapter `parse_url`). |
| `/items/<id>` | Full price-vs-time line chart (Chart.js), range selector 7d/30d/90d/all, stats for selected range (low/high/avg), recent history table, "Scrape now" button. |
| `/lists` | Create / rename lists; toggle tracked state; see item counts and last-updated per list. |
| `/logs` | Scrape run log with errors — the debugging surface for adapter regressions. |

## 9. Seed data — "High Demand Tech" list

Created in S1 as placeholders, verified & populated with real URLs in S7. Pick items that appear on ≥2 target retailers where possible:

| Category | Example candidates (verify at seed time) |
|---|---|
| SSDs | Samsung 990 Pro 2TB; WD Black SN850X 1–2TB |
| GPUs | NVIDIA RTX 4070 SUPER (AORUS/Gigabyte); AMD RX 7900 GRE |
| RAM | Corsair Vengeance DDR5-6000 32GB (2×16) CL30; Kingston Fury Beast DDR5 32GB |
| Flash storage | SanDisk Ultra Dual Drive USB-C 256GB; SanDisk High Endurance microSD 128GB |
| HDDs | Seagate IronWolf Pro 8TB; WD Red Plus 4TB (NAS drives, fits the TrueNAS theme) |

## 10. Docker / TrueNAS Dockage deployment (finalized in S8)

- `docker-compose.yml` at repo root: build `.`, map host port **8321 → container 8000**, volume `./data:/app/data`, env (`TZ`, `SCRAPE_INTERVAL_HOURS=6`, …), healthcheck on `/healthz`, `restart: unless-stopped`.
- On the NAS: project lives in the Dockage apps folder (e.g., `/mnt/<pool>/apps/pricehawk`). Sync from this machine via SSH/scp or rsync (no remote repo required; optional private GitHub later). In Dockage UI: point it at the compose file → create app → set env vars.
- Data survives upgrades/rebuilds in `./data`; back up by copying that one folder.

## 11. Git & commit conventions

- One logical change per commit; small increments within each session.
- Commit message format:

```
<imperative summary, ≤ ~70 chars>

[optional body]

Type: feature | fix | refactor | docs | chore | test | perf
```

- **End-of-session protocol (every session):** `pytest` green → `docker build` passes (from S0 onward) → `git status` clean → append a short entry to `SESSION_LOG.md` (what changed, open issues, notes for the next session).

## 12. Session plan (vibecoding handoffs)

**Protocol for every new session:**
1. `cd C:\Users\Trenton\coding-project\pricehawk`, run `git log --oneline -5`.
2. Read this PLAN.md §12 + your session's section, then the tail of `SESSION_LOG.md`.
3. Implement in small slices; run `pytest` after each slice.
4. Commit per logical change with the `Type:` footer (§11).
5. Finish only when: tests green, docker build passes, tree clean, SESSION_LOG updated.

### Session 0 — Repo scaffolding (Type: chore/docs)
**Goal:** runnable skeleton + tooling. **Deliverables:** project layout (`app/`, `tests/`), pinned deps (`requirements.txt` or pyproject), FastAPI app serving `/healthz`, pytest smoke test, Dockerfile + compose that build & run locally, `.gitignore`, README stub with dev instructions, `SESSION_LOG.md`.
**Done when:** `docker compose up --build` shows the healthcheck OK; `pytest` green; everything committed.

### Session 1 — Data layer (Type: feature)
**Goal:** SQLAlchemy models for lists/items/prices/scrape_runs, DB init with WAL, repository functions, seed CLI (`python -m app.cli seed`) that creates the "High Demand Tech" list with placeholder items from §9.
**Done when:** seeding creates the list + items; model CRUD covered by tests; pytest green.

### Session 2 — Scraper core + Newegg adapter (Type: feature)
**Goal:** `BaseSiteAdapter` interface, shared httpx client (§6), JSON-LD helper, `NeweggAdapter`, fixture-based tests, CLI `python -m app.cli scrape <url>` that stores a price point + `scrape_runs` row.
**Done when:** one live Newegg URL scrapes correctly by hand; all adapter tests pass offline from fixtures.

### Session 3 — Scheduler (Type: feature)
**Goal:** APScheduler job every `SCRAPE_INTERVAL_HOURS`, staggered per item, respects list toggles and the failure backoff (§6), writes prices/runs, updates `last_price/status/error_msg`.
**Done when:** with a short test interval, two items scrape automatically; toggling a list off pauses its items; pytest green.

### Session 4 — Dashboard UI + lists (Type: feature)
**Goal:** Jinja base layout; dashboard page with add-item form (URL → auto-detect), per-list sections with working toggle switches (POST endpoints), item rows with last price/status + simple sparkline; `/lists` management (create/rename).
**Done when:** can add a real URL from the UI, see it on the dashboard, and toggle its list on/off — all via browser.

### Session 5 — Charts + good-buy engine (Type: feature)
**Goal:** `/items/<id>` chart page (Chart.js line, range selector, stats), history JSON endpoint, good-buy scoring per §7 powering the prioritized top section, price-drop badges.
**Done when:** seeded items show real charts after a few scrape cycles; dashboard ordering visibly responds to price movement.

### Session 6 — Remaining adapters (Type: feature — several commits)
**Goal:** MicroCenterAdapter; BestBuyAdapter (JSON-LD/headers first, Playwright fallback only if blocked); AmazonAdapter best-effort or clearly-labeled stub with a note on the Keepa-API option; ServerPartsDeals forum-thread parser. Each adapter gets fixture tests + one live verification.
**Done when:** ≥4 adapters return correct prices via manual scrape; failures log cleanly without crashing the scheduler.

### Session 7 — Seed "High Demand Tech" + polish (Type: chore/docs)
**Goal:** verify & populate §9 items with real URLs across SSD/GPU/RAM/flash/HDD; run a couple of live cycles; UI polish (empty states, error banners, sort options); README usage docs.
**Done when:** dashboard shows the populated list; good-buy section behaves per §7 (including "insufficient data" handling).

### Session 8 — TrueNAS/Dockage deployment (Type: chore/feature)
**Goal:** final Dockerfile (non-root user, healthcheck), Dockage-tuned compose + `.env.example`, README "Deploy on TrueNAS via Dockage" steps (folder location, sync method, Dockage create-app flow); clean-room local `docker build` + smoke run.
**Done when:** the app runs from a fresh copy of the repo alone; the user then does the final manual NAS deploy in Dockage.

## 13. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Amazon/BestBuy bot walls | Fallback ladder per §6 (headers → JSON-LD → Playwright → third-party API); failures isolated via `scrape_runs`, never crash the scheduler |
| Site redesign breaks parsing | Fixture tests catch regressions; `status='error'` + last error message surfaced in UI/logs for quick triage |
| Cold start: no history yet → weak good-buy ranking | "Insufficient data" state until 7 days of real data (no faked stats); optional later enhancement: backfill via a third-party API |
| Retailer ToS (esp. Amazon) | Conservative rate limits, single-user local use, explicit disclaimer in README; Amazon kept optional/lowest-priority |
| Data loss on container rebuild | Single SQLite file in a bind volume; document backup = copy `./data` |

## 14. Milestones

- **M1 (after S3):** prices flow automatically for Newegg items.
- **M2 (after S5):** full dashboard with charts + good-buy ranking.
- **M3 (after S6–S7):** all retailers live + "High Demand Tech" list populated.
- **M4 (after S8):** running on TrueNAS via Dockage.

## 15. Configuration / env vars (all with sane defaults; documented in `.env.example` at S8)

| Var | Default | Meaning |
|---|---|---|
| `TZ` | system | Timezone for timestamps/scheduler |
| `SCRAPE_INTERVAL_HOURS` | 6 | Base scrape cadence per item (staggered) |
| `MIN_REQUEST_DELAY_S` | 2 | Min delay + jitter between requests to one site |
| `GOOD_BUY_PCT` | 12 | % below 90-day median required for good-buy badge |
| `DROP_ALERT_PCT` | 5 | % drop vs previous scrape flagged as a price drop |
| `DATA_DIR` | `/app/data` | SQLite location inside container |
| `LOG_LEVEL` | INFO | App log verbosity |

## 16. Suggested session prompts (copy-paste for each vibecoding session)

> Read `PLAN.md` §12 and the end of `SESSION_LOG.md`, then execute **Session N** exactly as specified — deliverables, done criteria, and commit conventions (§11, including the `Type:` footer). Do not start work from Session N+1.
