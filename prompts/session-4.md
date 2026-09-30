# Session 4 Prompt — Dashboard UI + Lists Management

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -25` — expected to end with Session 0–3 commits (scaffold, data layer, scraper + Newegg, scheduler). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — green before changes. Start the app (`uvicorn app.main:app --reload`) and confirm `/healthz` works with the scheduler running; note env vars from SESSION_LOG (e.g. `SCHEDULER_ENABLED`).

## Where we are
Sessions 0–3 are done: prices now flow **automatically** in the background for items whose list is active (Newegg only so far). Everything up to here has been headless — no HTML UI exists yet. Your job is the first user-facing surface: the dashboard and lists management, per **PLAN.md §8**.

## Your job this session (deliverables)

### 1. Template & static foundations
- `app/templates/base.html` — minimal clean layout: header with app name + nav links (**Dashboard**, **Lists**), content block. Keep it dependency-free (system font stack or one small hand-written CSS file).
- `app/static/css/app.css` — simple, readable styling; card/row layout for items; visual states for ok/error and dimmed/paused lists. No frontend frameworks.
- Wire Jinja2 into the FastAPI app (templates dir + static mount).

### 2. API endpoints (`app/api.py`, registered on the app)
- `POST /api/items/add` — JSON `{url, list_id?}`:
  - Unknown site → **400** with a clear message ("unsupported URL; supported sites: ...").
  - `parse_url` via registry; upsert item into target list (default: first active list if `list_id` omitted).
  - Attempt an immediate synchronous `fetch_price` so the user sees a price right away; **tolerate failure** — item is still added, with status=error + message. Return JSON of the resulting item (id, title, site, last_price_cents, status, error_msg).
- `GET /api/lists` → all lists with `active`, item count, and last-updated timestamp.
- `POST /api/lists` — create `{name}` (409/400 on duplicate).
- `POST /api/lists/{id}/toggle` — flip `active`; return updated list. (This is the tracked/untracked switch from PLAN §1.)
- `POST /api/lists/{id}/rename` — rename `{name}`.

### 3. Dashboard page (`GET /`)
Render per **PLAN.md §8**:
- **Top: "Good Buy Opportunities" section** — S5 implements the actual scoring, so for now render an explicit empty state card ("Tracking data is accumulating… good buys will appear here") in a container that's obviously where ranked deals will go. Do not fake data.
- **Add-item box**: URL input + optional list selector (populated from `/api/lists`) → POSTs to the add endpoint; show success/error message inline.
- **Per-list sections** for every list:
  - active lists: normal cards, each with a working toggle switch (`POST /api/lists/{id}/toggle`, then re-render).
  - untracked lists: rendered dimmed/collapsed with "Paused — tracking off" label and the same toggle to resume. (Toggling also pauses background scraping via S3's list gating.)
- **Item rows** within each section: title, site badge, last price formatted as `$x.xx` (or `—` if never scraped), status indicator (ok / error with tooltip or visible text of the last error message), and a tiny **sparkline** of the item's most recent ~20 prices.
- Add endpoint for the sparkline: `GET /api/items/{id}/sparkline` → JSON array of `{t, price_cents}` points (oldest→newest).

### 4. Lists page (`GET /lists`)
- Show all lists with name, active state, item count; create form; rename action per list; toggle controls — same endpoints as dashboard uses.

### 5. Tests
- `TestClient` API tests: add-item happy path (use the **injected fetch** pattern or a fixture-backed adapter so no network), unknown URL → 400, duplicate URL returns existing item, create/rename/toggle list flows, sparkline shape for an item with seeded prices.
- Page render tests: `/` and `/lists` return 200 with a couple of lists/items in the DB.

## Hard rules (apply to this and every session)
1. **Scope:** only this prompt's deliverables. No price chart page, no good-buy scoring (S5), no new adapters (S6). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **No network in tests** — inject/monkeypatch adapter fetches or use fixture-backed paths.
3. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
4. **Commits:** one logical change per commit; imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf`.
5. **PLAN.md is canonical.** Fix errors via a separate minimal `Type: docs` commit and call them out in your report.
6. **Blockers:** stop if stuck; record under "Open issues" with what you tried; leave the tree coherent — never end with unexplained half-work.
7. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] **Browser-verified flow:** paste a real Newegg product URL into the add box → it appears on the dashboard within seconds with its price (or an honest error status + message).
- [ ] Create a new list from `/lists`, rename one, toggle one off → its section dims and background scraping pauses for its items (confirm via `scrape_runs`/DB not growing), toggle back resumes.
- [ ] Sparklines render once an item has ≥2 price points; empty states are friendly, never broken HTML.
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Add Jinja base layout, static CSS, template wiring" — Type: feature
2. "Add lists API endpoints (create/rename/toggle)" — Type: feature
3. "Add add-item endpoint with immediate first scrape" — Type: feature
4. "Render dashboard with list sections and item rows + sparkline" — Type: feature
5. "Add lists management page" — Type: feature
6. "Cover API and page rendering with tests" — Type: test

## Finish protocol (mandatory)
1. Append a **Session 4** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 4 — <YYYY-MM-DD>
   - What was done: <bullets>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <none or bullets>
   - Notes for next session: <template/route structure S5 should extend, e.g. where the good-buy section hook lives>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, which live URL you added from the UI and what it showed, and confirmation that **Session 5** is ready (`prompts/session-5.md`).
