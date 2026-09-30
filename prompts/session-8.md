# Session 8 Prompt — Final Docker Image + TrueNAS/Dockage Deployment

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -45` — expected to end with Session 0–7 commits (scaffold, data layer, scraper + Newegg, scheduler, UI, charts/good-buy, remaining adapters, real seed data). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — green before changes. This session is about packaging and deployment, not features.

## Where we are
Sessions 0–7 are done: a complete, populated, locally-working single-container app (FastAPI + APScheduler + SQLite in one process; dashboard with good-buy ranking; per-item charts; adapters for all five sources). The Dockerfile/compose files exist but were S0 skeletons. Your job is to make this repo **deployable as-is on TrueNAS via Dockage** per **PLAN.md §13**: one container, host port 8321 → container 8000, single bind volume `./data:/app/data` (DB at `/app/data/pricewatch.db`).

## Your job this session (deliverables)

### 1. Final Dockerfile
- Keep `python:3.12-slim`; pin requirements install; copy app + tests out of the way or keep only what runtime needs (tests can stay — image size is not critical, but don't ship dev cruft you don't need).
- **Non-root user**: create a dedicated user (e.g. `useradd -r -s /sbin/nologin pricehawk`), chown `/app/data`, run as that user. SQLite + WAL files must be writable by it — verify, don't assume.
- **Healthcheck** in the Dockerfile: use Python's stdlib to avoid installing curl (`CMD ["python", "-c", "import urllib.request;urllib.request.urlopen('http://127.0.0.1:8000/healthz')"]`), with sensible `--interval/--timeout/--retries`.
- Keep any optional extras (e.g. the Playwright extra from S6, if it exists) **out of the default image** — document how to build/include it instead.

### 2. `.env.example` at repo root
Every variable an operator should know about, with comments and defaults, per **PLAN.md §15**: `TZ`, `SCRAPE_INTERVAL_HOURS=6`, `MIN_REQUEST_DELAY_S=2`, `GOOD_BUY_PCT=12`, `DROP_ALERT_PCT=5`, `DATA_DIR=/app/data`, `LOG_LEVEL=INFO` — plus anything later sessions introduced (e.g. `SCHEDULER_ENABLED`, `SCHED_TICKER_MINUTES`, `BESTBUY_USE_PLAYWRIGHT`) with clear comments on when to change them. It must be safe to copy verbatim into `.env`.

### 3. Finalized `docker-compose.yml` for Dockage
- Service `pricehawk`: `build: .`; ports `"8321:8000"`; volume `./data:/app/data`; environment from defaults with `${VAR:-default}` passthrough so it works **with or without** a `.env` file; `restart: unless-stopped`.
- Must work from a fresh clone of the repo with zero additional setup.

### 4. README "Deploy on TrueNAS via Dockage" section
Step-by-step for a non-expert:
1. Create folder `/mnt/<pool>/apps/pricehawk` (TrueNAS files UI or SSH).
2. Copy/sync the repo contents into it from this dev machine (`scp -r` / `rsync` over TrueNAS SSH; note that `.git`, `.venv`, and local `data/` are not needed on the NAS — the container creates its own volume data).
3. In Dockage UI: point at the compose file → Create App (note where env vars go if you want to override defaults, e.g. `TZ`).
4. Open `http://<nas-ip>:8321` — verify health + dashboard.
5. **Backup/restore**: the entire app state is the `./data` folder (SQLite DB at `/app/data/pricewatch.db`) — copying that folder is a full backup; stopping the container before copy is safest.

### 5. Clean-room verification (the real gate for this session)
Using Docker locally if available:
- Build from a **fresh copy** of the repo in a temp dir (simulate what's on the NAS): `docker build` → run with a fresh temp volume → `/healthz` returns ok within ~30 s.
- Seed via container CLI (`docker compose exec pricehawk python -m app.cli seed`) and confirm the DB file appears under the mounted data dir.
- `docker compose down && docker compose up` again → seeded data survives (volume persistence proof).
If Docker is **not** available in this environment, do everything you can statically (compose validation with `docker compose config` if present, or careful review), and mark the clean-room gate as "skipped — no local Docker; run on NAS" under Open issues.

## Hard rules (apply to this and every session)
1. **Scope:** packaging + deployment docs only. No new features, no UI changes beyond what's needed for a correct build. Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **Don't break the dev workflow:** local venv/uvicorn runs must still work exactly as before; defaults in `.env.example` must match PLAN.md §15 and current code behavior.
3. **Gates before finishing:** `pytest` green; clean-room docker verification per above (or documented skip); `git status` clean.
4. **Commits:** one logical change per commit — Dockerfile hardening, `.env.example`, compose finalization, README deploy section, and the SESSION_LOG update each get their own commit. Imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** If §13/§15 details conflict with what you find in the code, prefer what makes a correct deployment and record the discrepancy; a PLAN.md correction itself only gets its own `Type: docs` commit if it's actually wrong.
6. **Blockers:** stop if stuck (e.g., Docker unavailable); record under "Open issues" precisely what still needs to be verified on the NAS, with exact commands. Leave the tree coherent — never end with unexplained half-work.
7. Never commit `.venv/`, `data/`, `.env` (only `.env.example`), or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] Fresh-copy build → healthy container on port 8321 with `/healthz` ok; non-root process confirmed (`docker inspect ... User` or `ps` inside).
- [ ] DB created at `/app/data/pricewatch.db` under the bind volume and **survives a down/up cycle**.
- [ ] `.env.example` complete, commented, safe to copy verbatim.
- [ ] README deploy steps accurate end-to-end for TrueNAS + Dockage (folder → sync → create app → browse → backup).
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; pytest green.

## Suggested commit order (adapt as needed)
1. "Harden Dockerfile with non-root user and healthcheck" — Type: chore
2. "Add .env.example documenting all runtime settings" — Type: docs
3. "Finalize compose file for Dockage deployment" — Type: chore
4. "Document TrueNAS/Dockage deployment in README" — Type: docs

## Finish protocol (mandatory)
1. Append a **Session 8** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 8 — <YYYY-MM-DD>
   - What was done: <bullets incl. clean-room verification results or skip reason>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <anything that must be verified on the real NAS, with exact commands>
   - Notes for next session: project is deploy-ready; remaining work is manual NAS-side deployment + any open adapter issues from S6
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, clean-room evidence (or what's left for the NAS), and that **all planned build sessions are complete** — next step is the manual TrueNAS/Dockage deployment from the README.
