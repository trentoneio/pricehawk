# Session 1 Prompt — Data Layer (DB, Models, Seed CLI)

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -10` — expected to end with Session 0 commits (FastAPI scaffold, pinned deps + health test, Dockerfile/compose skeleton, README/SESSION_LOG). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: create venv (`python -m venv .venv && source .venv/bin/activate`), install deps, run `pytest` — must be green **before** you change anything.

## Where we are
Session 0 is done: the repo has a runnable FastAPI skeleton (app factory + `/healthz`), pinned dependencies, tests, Dockerfile/compose skeletons, README, and SESSION_LOG.md. Your job is to build the persistence layer that everything else reads/writes.

## Your job this session (deliverables)

### 1. Dependencies
- Add **SQLAlchemy 2.x** (pinned) to `requirements.txt`. Nothing else new needed.

### 2. `app/db.py`
- Engine creation from the configured `DATA_DIR` (create the directory if missing); a session factory; an `init_db()` function that creates all tables.
- SQLite pragmas on connect: `journal_mode=WAL` and a sane `busy_timeout`.
- Wire `init_db()` into the FastAPI lifespan in `app/main.py` so any app start guarantees tables exist (dev runs use whatever `DATA_DIR` is set to — make sure a local dev path works, e.g. honor an env override).

### 3. `app/models.py` — SQLAlchemy 2.0 style (`DeclarativeBase`, `Mapped[...]`)
Per **PLAN.md §5**, four tables:
```sql
lists        (id PK, name UNIQUE NOT NULL, active BOOL default true, created_at)
items        (id PK, list_id FK→lists, url UNIQUE NOT NULL, site TEXT, title TEXT, sku NULL,
              added_at, last_price_cents INT NULL, last_scraped_at NULL, status TEXT 'ok'|'error', error_msg TEXT)
prices       (id PK, item_id FK→items, price_cents INT NOT NULL, in_stock BOOL NOT NULL,
              source_url TEXT, scraped_at NOT NULL)   -- index on (item_id, scraped_at)
scrape_runs  (id PK, item_id FK→items, started_at NOT NULL, finished_at NULL,
              outcome TEXT 'ok'|'price_changed'|'error', detail TEXT)
```
- Use a non-shadowing class name for the `lists` table (e.g. `WatchList` → table `lists`).
- Add relationships where they help (item ↔ list, item ↔ prices).

### 4. `app/repo.py` — repository functions
Small function set that later sessions (scheduler, API) will call instead of fiddling with sessions directly:
- `create_list(name) -> WatchList` (error/raise on duplicate name)
- `list_lists() -> list[WatchList]`
- `get_item_by_url(url)` / `add_item(list_id, url, site, title, sku=None)` — **idempotent**: if the URL already exists, return the existing item without duplicating.
- `record_price(item_id, price_cents, in_stock, source_url, scraped_at=now)` — append a `prices` row AND update the item's `last_price_cents`, `last_scraped_at`, `status='ok'`, clear `error_msg`. Detect "price changed" vs same-price for the caller to use when logging runs.
- `record_run(item_id, started_at, finished_at, outcome, detail=None)` — append a `scrape_runs` row.
- `set_item_error(item_id, error_msg)` — set `status='error'`, store the message.

### 5. `app/cli.py` — CLI entrypoint (`python -m app.cli ...`)
- Subcommand **`seed`**: create the **"High Demand Tech"** list if missing, then populate it with placeholder items per **PLAN.md §9** (SSDs, GPUs, RAM, flash storage, HDDs — roughly 10 placeholders total). Placeholder convention: `url = "pending://<category>/<slug>"`, `site = "unknown"`, descriptive titles. Must be **idempotent** (re-running adds nothing new) and print what it created vs skipped.
- Design the CLI so later sessions will add subcommands (`scrape` in S2, etc.) — keep argument parsing extensible.

### 6. Tests (`tests/test_models.py` or similar)
- Use a temp SQLite file (fixture), not the real `DATA_DIR`.
- Cover: list creation + duplicate-name behavior; idempotent item add by URL; price recording updates item state and appends to `prices`; run logging; seed command idempotency (run twice).

## Hard rules (apply to this and every session)
1. **Scope:** only this prompt's deliverables. No scraping code, no scheduler, no UI — those are S2/S3/S4+. Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **Gates before finishing:** `pytest` green (old + new tests); `docker build -t pricehawk .` passes **if Docker is available** (else note it in SESSION_LOG.md); `git status` clean.
3. **Commits:** one logical change per commit; imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = one of `feature | fix | refactor | docs | chore | test | perf`.
4. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
5. **Blockers:** stop if stuck; record under "Open issues" in SESSION_LOG.md with what you tried; leave the tree coherent — never end with unexplained half-work.
6. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] `python -m app.cli seed` creates "High Demand Tech" + placeholder items; running it a second time changes nothing.
- [ ] Starting the app initializes the DB automatically (lifespan hook works).
- [ ] New tests pass; pre-existing health test still passes.
- [ ] Docker build passes (if available); all commits carry `Type:` footers; SESSION_LOG updated.

## Suggested commit order (adapt as needed)
1. "Add SQLAlchemy models and DB init with WAL" — Type: feature
2. "Add repository functions for lists, items, prices, runs" — Type: feature
3. "Add seed CLI creating High Demand Tech placeholders" — Type: feature
4. "Cover data layer with model/CLI tests" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 1** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 1 — <YYYY-MM-DD>
   - What was done: <bullets>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets>
   - Notes for next session: <anything S2 should know, e.g. repo function signatures that changed>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, any repo-function API decisions S2 should rely on, and confirmation that **Session 2** is ready (`prompts/session-2.md`).
