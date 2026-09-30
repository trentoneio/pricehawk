# Session 6 Prompt — Remaining Site Adapters (Micro Center, Best Buy, Amazon, Server Parts Deals)

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -35` — expected to end with Session 0–5 commits (scaffold, data layer, scraper + Newegg, scheduler, UI, charts + good-buy). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — green before changes. Study `app/scrapers/newegg.py` as the **reference implementation**: it shows the base contracts, the injectable fetch callable (tests depend on it), JSON-LD-first parsing, and registry wiring in `app/scrapers/__init__.py`.

## Where we are
Sessions 0–5 are done: a working single-container app with automatic background scraping, dashboard + good-buy ranking, per-item charts — but only **Newegg** actually scrapes. Your job is the remaining four site adapters from **PLAN.md §6**, each as its own commit, each with fixture tests and one live verification attempt.

## Strategy per site (from PLAN.md §6 — follow these priorities)

### 1. `microcenter.py` — Micro CenterAdapter
- Static Magento-style HTML + JSON-LD on product pages; low–medium difficulty.
- `parse_url`: recognize microcenter.com product URLs; title from JSON-LD name or `<title>`; sku if present in the page (optional).
- `fetch_price`: shared client → GET page → **JSON-LD offers first**, DOM fallback for price/availability; in-stock from availability text. In-store stock info is a nice-to-have later — do not scope it here.

### 2. `bestbuy.py` — BestBuyAdapter
- Expected to be Akamai-protected: send strong desktop headers (UA, Accept, Accept-Language) and try **JSON-LD / embedded page state first**.
- If plain GETs consistently fail (403 / challenge HTML), implement an optional headless fallback with Playwright behind env `BESTBUY_USE_PLAYWRIGHT=1` — add playwright as a separate extra (`requirements-playwright.txt` or clearly marked extra in requirements) so the default image stays small and tests stay fast. Import it lazily; never import at module load when disabled.
- Whatever the outcome, document it honestly: working adapter, partial (works with Playwright), or blocked → clean `ScrapeError` the UI can display.

### 3. `amazon.py` — AmazonAdapter (best-effort, lowest priority)
- Amazon's ToS prohibits scraping — keep this clearly optional/best-effort per PLAN.md §6.
- Acceptable outcomes, in order of preference: (a) occasionally works with plain headers + JSON-LD; (b) a **labeled stub** where `parse_url` fully works and `fetch_price` raises `ScrapeError("Amazon scraping is unreliable/blocked in v1 — see PLAN.md §6")`.
- Add a short README note: the intended future path is a third-party price API such as Keepa behind this same adapter interface.

### 4. `serverpartsdeals.py` — ServerPartsDealsAdapter (forum, supplemental)
- Deal-thread parser: item URL = a deal thread on serverpartsdeals.com.
- `parse_url`: title from the thread title; site name as-is; no sku.
- `fetch_price`: GET the thread → scan post bodies (first few posts / OP) for price patterns (`$1,234`, `$99.00`) associated with the product in the thread title; return a `ScrapeResult` with that price and `in_stock=True` (a listed deal implies availability); if no price can be found → `ScrapeError("no price found in thread")`. Keep expectations modest — this source is supplemental.

### Common to all four
- Register each adapter in `app/scrapers/__init__.py` as it lands; update the "supported sites" message used by the API/CLI if one hard-codes it (derive from the registry instead, where easy).
- Each adapter must work with an **injected fetch callable** exactly like Newegg does — fixture tests are mandatory.
- Save 1–2 real pages per working site under `tests/fixtures/<site>/` (be polite: only a handful of live requests total this session; everything else via fixtures). Commit the fixtures.
- The scheduler must never crash on adapter failure — verify by adding an item from a failing/blocked site and confirming it lands status=error with a clean message in `scrape_runs`, while other items keep scraping.

## Hard rules (apply to this and every session)
1. **Scope:** only these four adapters (+ their fixtures/tests/registry wiring). No UI redesign, no seed-data population (S7), no deployment work (S8). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **Be a good citizen of the web:** strictly limit live requests to what verification needs; repeated testing always goes through fixtures.
3. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
4. **Commits:** one logical change per commit — plan on at least one commit per adapter, plus a fixtures/tests commit (or fold tests into each adapter's commit). Imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
6. **Blockers:** bot-walled sites are an *expected* outcome, not a failure of the session — implement the fallback ladder as far as reasonable, document exactly what works/doesn't per site under "Open issues"/notes, keep the code complete per spec with passing fixture tests, and move on. Leave the tree coherent.
7. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] **≥ 4** of {Newegg, Micro Center, Best Buy, Server Parts Deals} return correct prices via manual CLI scrape (`python -m app.cli scrape <url>`). Amazon is optional/stub and doesn't count toward the four.
- [ ] Each adapter has fixture-based offline tests; `pytest` green overall.
- [ ] At least one item from a failing site shows status=error with its message in the dashboard/DB, and background scraping of *other* items continued uninterrupted (check `scrape_runs`).
- [ ] Per-site outcome table recorded in SESSION_LOG.md (works / works-with-playwright / blocked+stubbed).
- [ ] All commits carry `Type:` footers; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Add Micro Center adapter with JSON-LD parsing" — Type: feature
2. "Add Best Buy adapter with headers and optional Playwright fallback" — Type: feature
3. "Add Amazon best-effort adapter / labeled stub per plan §6" — Type: feature
4. "Add Server Parts Deals forum-thread parser" — Type: feature
5. "Add fixtures and offline tests for new adapters" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 6** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 6 — <YYYY-MM-DD>
   - What was done: <bullets>
   - Per-site outcomes: Newegg …, Micro Center …, Best Buy …, Amazon …, SLD …
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <blocked sites + what was tried + env vars like BESTBUY_USE_PLAYWRIGHT>
   - Notes for next session: <which URLs/adapters S7 should trust when seeding real items>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, the per-site outcome table with one verified live URL per working site, and confirmation that **Session 7** is ready (`prompts/session-7.md`).
