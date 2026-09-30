# Session 3 Prompt — Background Scheduler

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -20` — expected to end with Session 0–2 commits (scaffold, data layer, scraper core + Newegg adapter). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — must be green before you change anything.

## Where we are
Sessions 0–2 are done: FastAPI app with `/healthz`; DB layer (lists/items/prices/scrape_runs) in `app/db.py`, `app/models.py`, `app/repo.py`; scraper framework (`app/scrapers/` with base contracts, shared rate-limited http client, JSON-LD helper) and a working **NeweggAdapter**; CLI supports `seed` and `scrape <url>`. Scraping currently only happens when you run the CLI by hand. Your job is to make prices flow automatically in the background.

## Design decision (make it explicit and document it)
Implement a **single ticker job** rather than one job per item:
- APScheduler `BackgroundScheduler` with one interval job every **N minutes** (`SCHED_TICKER_MINUTES`, default 5, env-configurable).
- Each tick scans for items that are due now: their list is active AND the item isn't backed off AND `next_due_at <= now`. Scrape them sequentially (the shared http client already enforces per-host delays + jitter).
This keeps backoff/toggle logic in one place and makes tests trivial. Record this choice in SESSION_LOG.md under "Notes for next session".

### Schema addition required
`items` needs two new columns: `next_due_at` (datetime, default = now on creation → first scrape happens at the next tick) and `fail_count` (int, default 0). SQLite has no migrations framework here — add a small idempotent "ensure columns" step in `app/db.py` (`ALTER TABLE items ADD COLUMN ...` guarded by a column-existence check). Do this before anything else so app startup stays safe.

## Your job this session (deliverables)

### 1. Shared scrape path
Refactor the CLI's `scrape <url>` logic into one reusable function, e.g. `app/scrapers/service.py::scrape_item(item) -> ScrapeResult | raises/handles`, that:
- resolves the adapter for the item URL (missing adapter → record error run + set_item_error, do not crash),
- calls `fetch_price`, records a price point and `scrape_runs` row via repo functions (`ok` / `price_changed` / `error` with detail message).
The CLI subcommand becomes a thin wrapper over this function. The scheduler uses the same function — **one code path** for manual and automatic scrapes.

### 2. `app/scheduler.py`
- Build/start/stop an APScheduler `BackgroundScheduler`.
- Tick job logic per design above; after each item scrape:
  - success → `fail_count = 0`, `next_due_at = now + SCRAPE_INTERVAL_HOURS + random jitter(0..30 min)`, status ok.
  - failure → `fail_count += 1`, `next_due_at = now + SCRAPE_INTERVAL_HOURS * min(2 ** fail_count, 8)` (cap ≈ 4 days of doubling), item status `error` with the last message. Items that have no adapter or a `pending://` placeholder URL must simply be skipped silently-ish (count them, don't spam runs) — those are S7's job to fix.
- Wire into the FastAPI lifespan in `app/main.py`: start on startup, shut down cleanly on exit. Add an env/config flag to disable the scheduler (e.g. `SCHEDULER_ENABLED` default true; tests and manual runs can turn it off).

### 3. Tests (`tests/test_scheduler.py`)
- Monkeypatch adapter fetches (injected callable) to return canned results or raise — no network in tests.
- Cover: only items whose list is active are scraped; a tick with nothing due does nothing; success advances `next_due_at` and resets fail_count; failure doubles the delay and caps it; error status/message recorded; placeholder/adapterless items don't crash the tick.
- Drive ticks manually (call the job function directly) with controlled "now" values rather than real sleeps — except **one** short integration test that uses a 1-second interval for a couple of seconds if APScheduler makes it easy.

## Hard rules (apply to this and every session)
1. **Scope:** only this prompt's deliverables. No UI work (S4), no other adapters (S6). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **No network in tests.** All adapter interaction is injected/monkeypatched.
3. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
4. **Commits:** one logical change per commit; imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
6. **Blockers:** stop if stuck; record under "Open issues" with what you tried; leave the tree coherent — never end with unexplained half-work.
7. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] Starting the app begins background scraping: with a short test interval (e.g. ticker 1–2 min, SCRAPE_INTERVAL_HOURS overridden tiny via env), **two real Newegg items scrape automatically** within ~2 minutes without any CLI use — verify by new rows in `prices`/`scrape_runs`.
- [ ] Setting a list's `active=false` pauses its items (no new run rows for them on subsequent ticks); flipping it back resumes.
- [ ] A failing item backs off with doubling delay, shows status=error + message, and never crashes the scheduler loop.
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Add next_due_at/fail_count columns with idempotent schema step" — Type: feature
2. "Extract shared scrape_item service used by CLI and scheduler" — Type: refactor
3. "Add APScheduler ticker job with backoff and list gating" — Type: feature
4. "Wire scheduler into app lifespan with enable flag" — Type: feature
5. "Cover scheduler behavior with offline tests" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 3** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 3 — <YYYY-MM-DD>
   - What was done: <bullets, incl. the ticker design decision + how it's disabled in tests>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets>
   - Notes for next session: <env vars S4/S5 should know, e.g. SCHEDULER_ENABLED, SCHED_TICKER_MINUTES>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, evidence of the live auto-scrape (which items got new price rows), and confirmation that **Session 4** is ready (`prompts/session-4.md`).
