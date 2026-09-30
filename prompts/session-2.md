# Session 2 Prompt — Scraper Core + Newegg Adapter

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -15` — expected to end with Session 0 (scaffold) and Session 1 (data layer) commits. If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — must be green before you change anything.

## Where we are
Sessions 0–1 are done: FastAPI skeleton with `/healthz`; SQLAlchemy models (`lists`, `items`, `prices`, `scrape_runs`) in `app/models.py`; DB init with WAL in `app/db.py`; repository functions in `app/repo.py`; CLI with a `seed` subcommand that creates the "High Demand Tech" list. **No scraping exists yet.** Your job is the scraper foundation plus the first real retailer (Newegg).

## Your job this session (deliverables)

### 1. Package layout
Create `app/scrapers/` containing:
- `__init__.py` — a **registry** mapping site name → adapter, and a helper `adapter_for(url)` that detects the site from the URL host (`newegg.com`, `microcenter.com`, `bestbuy.com`, `amazon.com`, `serverpartsdeals.com`) and returns the adapter or `None`. Only Newegg is registered today; later sessions add adapters to this registry — keep it trivially extensible.
- `base.py` — shared contracts (see below).
- `http_client.py` — shared HTTP layer (see below).
- `jsonld.py` — JSON-LD helper (see below).
- `newegg.py` — the Newegg adapter.

### 2. `app/scrapers/base.py`
```python
class ScrapeError(Exception): ...            # carries a human-readable message; all adapter failures raise this
@dataclass ParsedItem: site, title, sku (optional), url
@dataclass ScrapeResult: price_cents:int, in_stock:bool, source_url:str

class BaseSiteAdapter(abc.ABC):
    site_name: str
    host_patterns: tuple[str, ...]           # e.g. ("newegg.com",)
    def parse_url(self, url: str) -> ParsedItem: ...        # pure-ish; may fetch page for title if needed
    def fetch_price(self, item: ParsedItem) -> ScrapeResult: ...
```
**Design requirement:** adapters must accept an injectable fetch callable (e.g. constructor arg `fetch=None` defaulting to the shared client). This is what makes fixture-based offline tests possible — every test in this repo will rely on it.

### 3. `app/scrapers/http_client.py` — shared client
A wrapper around one `httpx.Client`:
- Rotating pool of 3–5 realistic desktop User-Agent strings (pick per request).
- Per-host minimum delay **plus jitter** based on env `MIN_REQUEST_DELAY_S` (default 2s) between consecutive requests to the same host.
- Timeouts (~15 s), and up to **3 retries with exponential backoff** for network errors, HTTP 429, and 5xx; then raise `ScrapeError` with a clear message.
- Returns response text (and status if needed). This is the only module that talks raw HTTP — adapters never build their own clients.

### 4. `app/scrapers/jsonld.py`
Helper(s) to: extract all `script[type="application/ld+json"]` blocks from an HTML string; find Product nodes; read offers → price + availability. Adapters should try JSON-LD first, DOM parsing second (per **PLAN.md §6**).

### 5. `app/scrapers/newegg.py` — NeweggAdapter
- `parse_url`: accept Newegg product URLs (`https://www.newegg.com/<slug>-<PartNumber>/Product/-<PartNumber>.sku` and common variants); extract the Part Number as `sku`; derive a clean title (from JSON-LD name if a fetch is justified, else from the URL slug).
- `fetch_price`: GET the product page → **JSON-LD offers first** (`offers.price`, availability) → DOM fallback for the price element and in-stock text. Convert to cents; set `in_stock` from availability signals ("In Stock" etc.). Raise `ScrapeError` with a useful message on failure (page gone, no price found, blocked).

### 6. CLI extension
Add subcommand **`scrape <url> [--list NAME]`** to `app/cli.py`:
- Resolve adapter via registry; unknown site → clear error message listing supported sites.
- `parse_url`, upsert item into the target list (default: "High Demand Tech"; create if missing only when explicitly requested — otherwise require an existing list or a `--create-list` flag).
- `fetch_price`, then use repo functions to record the price and a `scrape_runs` row (`ok`, `price_changed`, or `error`). Print a one-line human summary (title, old→new price, in-stock, status).

### 7. Fixture tests
- During this session, **download 1–2 real Newegg product pages** (be polite: only a couple of requests) and save them under `tests/fixtures/newegg/` (commit the fixtures — they are your regression net; note any ToS caveats in a README line for the tests dir).
- Tests call `parse_url` / `fetch_price` against fixture text via the injected fetch callable: correct price/cents, in-stock detection, and clean `ScrapeError` on a "no price" or blocked fixture (add one small synthetic error fixture if real pages don't cover it).

## Hard rules (apply to this and every session)
1. **Scope:** only this prompt's deliverables. No scheduler (S3), no UI (S4), no other adapters (S6). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **Be a good citizen of the web:** live requests are limited to what verification needs (a few at most). All repeated testing goes through fixtures.
3. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
4. **Commits:** one logical change per commit; imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
6. **Blockers:** if Newegg blocks even basic requests, don't fight it for hours — record what you tried under "Open issues", leave the adapter code complete per spec with fixture tests passing, and note that live verification failed. Leave the tree coherent.
7. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] `python -m app.cli scrape <real Newegg URL>` stores a correct price + run row in the DB and prints a sane summary.
- [ ] Fixture-based tests pass **offline** (no network); they would catch regressions if Newegg's HTML changes.
- [ ] Unknown URLs are rejected with a helpful message; failures raise/handle `ScrapeError` cleanly (nothing crashes).
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Add scraper base contracts and shared http client" — Type: feature
2. "Add JSON-LD extraction helper" — Type: feature
3. "Add Newegg adapter with registry wiring" — Type: feature
4. "Add scrape CLI subcommand storing price points" — Type: feature
5. "Add Newegg fixtures and offline adapter tests" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 2** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 2 — <YYYY-MM-DD>
   - What was done: <bullets>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets, e.g. live-block details if any>
   - Notes for next session: <adapter interface + repo functions S3 must reuse verbatim>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, **which live URL you verified** and its result, plus confirmation that **Session 3** is ready (`prompts/session-3.md`).
