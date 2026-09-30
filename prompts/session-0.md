# Session 0 Prompt — Repo Scaffolding & Tooling

**Project:** PriceHawk — local, self-hosted tech price tracker (single Docker container, deployed on TrueNAS via Dockage).
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** the shell for this session may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`). Detect with `ls /c` vs `ls /mnt/c`; use whichever exists.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository itself and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -8` — expected history: commit #1 "Add project plan..." (Type: docs), commit #2 "Relocate repo to coding-projects..." (Type: chore). If it doesn't match, **stop and report** before doing anything.
   - `git status` — must be clean before you begin.
   - Read `PLAN.md` in full (it is the canonical requirements document), then read this file.
2. Confirm tooling available: Python 3.10+ (`python --version`), git, and whether Docker exists (`docker version`). Record availability in your final report.

## Where we are
This is the **first build session**. The repo contains only `PLAN.md`, `.gitignore`, and `prompts/`. Nothing else exists yet — you are creating the runnable skeleton that every later session builds on.

## Your job this session (deliverables)

### 1. Python package `app/`
- `app/__init__.py`
- `app/config.py` — load configuration from environment variables with defaults, per **PLAN.md §15**: `TZ`, `SCRAPE_INTERVAL_HOURS` (6), `MIN_REQUEST_DELAY_S` (2), `GOOD_BUY_PCT` (12), `DROP_ALERT_PCT` (5), `DATA_DIR` (`/app/data` for container use; allow local override via env in dev), `LOG_LEVEL` (INFO). A small plain module reading `os.getenv` is fine — no need for a settings framework.
- `app/main.py` — FastAPI app factory: `def create_app() -> FastAPI`. For now it exposes one route: `GET /healthz` → JSON `{"status": "ok", "service": "pricehawk"}`. Later sessions will extend this with lifespan hooks (DB init in S1, scheduler in S3) — design the factory so that's easy to add.

### 2. Dependencies & tests
- `requirements.txt` at repo root with **pinned** versions (`==`) of: fastapi, uvicorn[standard], jinja2, httpx, pytest (SQLAlchemy and APScheduler are added by later sessions — don't pre-add them).
- `tests/test_health.py` — use FastAPI's `TestClient`; assert `/healthz` returns 200 and `"status": "ok"`.

### 3. Docker skeleton
- `Dockerfile`: base `python:3.12-slim`; copy requirements first (layer caching), `pip install`, copy `app/`, `ENV DATA_DIR=/app/data`, `EXPOSE 8000`, `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]`. Note in a comment that S8 will harden this (non-root user, healthcheck).
- `docker-compose.yml` at repo root: service `pricehawk`, `build: .`, ports `"8321:8000"`, volume `./data:/app/data`, environment passthrough (`TZ=${TZ:-UTC}`, etc.), `restart: unless-stopped`. This is a skeleton; S8 finalizes it for Dockage.

### 4. Docs
- `README.md` stub: one paragraph on what the project is (link to `PLAN.md`), then "Local development" quickstart (create `.venv`, install requirements, run uvicorn with reload), "Run tests" (`pytest`), and "Docker" (`docker compose up --build`). Note that runtime data lives in `./data`.
- `SESSION_LOG.md`: short header explaining the entry format used by every session, then your Session 0 entry (see Finish protocol).

## Hard rules (apply to this and every session)
1. **Scope:** implement only what this prompt lists. Do not start S1+ work (no DB models, no scrapers). Small incidental fixes are fine if committed separately with their own `Type:` footer.
2. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes **if Docker is available** in this environment (otherwise note "skipped — docker unavailable" in SESSION_LOG.md); `git status` clean.
3. **Commits:** one logical change per commit. Message = imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` followed by one of `feature | fix | refactor | docs | chore | test | perf`.
4. **PLAN.md is canonical.** If you find an error in it, make a minimal correction as its own commit with `Type: docs` and call it out in your report — don't silently rewrite it.
5. **Blockers:** if stuck on something outside your control, stop; record details under "Open issues" in SESSION_LOG.md (what you tried, what failed); leave the tree coherent (committed or cleanly rolled back) — never end with unexplained half-work.
6. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`; double-check with `git status`).

## Definition of done (checklist)
- [ ] `pytest` passes, including the `/healthz` smoke test.
- [ ] `uvicorn app.main:app` serves `/healthz` locally in a venv.
- [ ] If Docker available: `docker compose up --build` starts and `http://localhost:8321/healthz` returns ok (then bring it down).
- [ ] README, `.gitignore`, and SESSION_LOG.md present; all commits carry the `Type:` footer.

## Suggested commit order (adapt as needed)
1. "Scaffold FastAPI app with /healthz endpoint" — Type: chore
2. "Pin dependencies and add health smoke test" — Type: test
3. "Add Dockerfile and compose skeleton" — Type: chore
4. "Document quickstart in README; init SESSION_LOG.md" — Type: docs

## Finish protocol (mandatory)
1. Append a **Session 0** entry to `SESSION_LOG.md` using this exact shape:
   ```markdown
   ## Session 0 — <YYYY-MM-DD>
   - What was done: <bullets>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets>
   - Notes for next session: <anything S1 should know>
   ```
2. Commit the log update as its own commit — `Type: docs`.
3. Report to the user: list of commits made, test/build results, whether Docker was available, and confirmation that **Session 1** is ready (`prompts/session-1.md`).
