# Session 7 Prompt — Seed "High Demand Tech" with Real Items + Polish

**Project:** PriceHawk — local, self-hosted tech price tracker.
**Repo root:** `C:\Users\Trenton\coding-projects\pricehawk`
**Shell paths:** may be Git Bash/MSYS2 (`/c/Users/Trenton/coding-projects/pricehawk`) or WSL (`/mnt/c/Users/Trenton/coding-projects/pricehawk`) — detect with `ls /c` vs `ls /mnt/c`.

You are starting fresh with **no memory of previous sessions**. The only prior state is this repository and this `prompts/` directory.

## Start here (do these first, in order)
1. `cd` into the repo root. Run:
   - `git log --oneline -40` — expected to end with Session 0–6 commits (scaffold, data layer, scraper + Newegg, scheduler, UI, charts/good-buy, remaining adapters). If not, **stop and report**.
   - `git status` — must be clean.
   - Read `PLAN.md` in full (canonical requirements), then the tail of `SESSION_LOG.md`, then this file.
2. Baseline: venv + install deps, run `pytest` — green before changes; start the app and confirm `/healthz` and `/` render. Inspect the DB for the S1 placeholder items (`url LIKE 'pending://%'`) in the "High Demand Tech" list — those are what you're replacing this session.

## Where we are
Sessions 0–6 are done: a complete single-container app with background scraping, dashboard + good-buy ranking, per-item charts, and adapters for all five sources (some possibly partial/blocked — check SESSION_LOG.md's per-site outcome table). The "High Demand Tech" list still contains **S1 placeholders** (`pending://...` URLs) that don't scrape. Your job: make it real, run live cycles, and polish the UX + docs.

## Your job this session (deliverables)

### 1. Replace placeholders with real, verified items
Per **PLAN.md §9**, populate "High Demand Tech" with roughly **8–12 real products** across all five categories:
- **SSDs:** e.g. Samsung 990 Pro 2TB; WD Black SN850X 1–2TB
- **GPUs:** e.g. NVIDIA RTX 4070 SUPER (AORUS/Gigabyte); AMD RX 7900 GRE
- **RAM:** e.g. Corsair Vengeance DDR5-6000 32GB (2×16) CL30; Kingston Fury Beast DDR5 32GB
- **Flash storage:** e.g. SanDisk Ultra Dual Drive USB-C 256GB; SanDisk High Endurance microSD 128GB
- **Hard drives:** e.g. Seagate IronWolf Pro 8TB; WD Red Plus 4TB

Rules:
- Prefer items sold at ≥ 2 supported retailers (one item row per URL — add the same product from a second retailer as its own row when useful).
- Pick whichever adapters SESSION_LOG.md says are working for each site; use `python -m app.cli scrape <url>` to **verify every single item** returns a valid price before you count it done. Fix or swap any URL that fails (different retailer, different product variant) — do not leave dead rows in the seed list.
- Replace placeholder rows rather than accumulating duplicates: update their `url`/`title`/`sku`/`site` in place (a small CLI helper like `seed --real` or a one-off script is fine — keep it committed if you make one).

### 2. Run live cycles
- Temporarily set a short interval via env (`SCRAPE_INTERVAL_HOURS=0.1`, ticker minutes low) so the scheduler completes **at least two full scrape cycles** over the real items while you polish; then restore defaults for the final state. Record what you observed (prices stored, any adapter errors).

### 3. UI polish (small, targeted changes only — no redesign)
- Empty states everywhere a list/section can be empty ("No items yet — add one above", "Insufficient data" per good-buy rules).
- Item rows with status=error show the last error message visibly (or as tooltip), not just an icon.
- Per-list-section sort dropdown: deal score / price / name (client-side over rendered rows is acceptable at this scale).

### 4. README usage docs
Expand `README.md` beyond quickstart: how to add items by URL; what tracked/untracked lists do and how toggling pauses scraping; supported-sites table with honest caveats (from S6's outcome table, incl. Amazon/Best Buy notes); good-buy badge meanings + thresholds (`GOOD_BUY_PCT`, `DROP_ALERT_PCT`) pointing at PLAN.md §7.

## Hard rules (apply to this and every session)
1. **Scope:** seed data replacement, live-cycle verification, the listed UI polish, README docs. No new features (no alerts, no new endpoints beyond what polish needs), no deployment work (S8). Small incidental fixes fine if committed separately with their own `Type:` footer.
2. **Be a good citizen of the web:** verify each real URL with at most one manual scrape during this session; let the short-interval cycles do the rest within normal rate limits — never add extra delay-bypassing code to "get more data".
3. **No fake data.** Every price in the DB after this session must come from a real scrape (or be pre-existing S1–S5 data). Do not hand-insert prices.
4. **Gates before finishing:** `pytest` green; `docker build -t pricehawk .` passes if Docker is available (else note in SESSION_LOG.md); `git status` clean.
5. **Commits:** one logical change per commit — seed-data commits are fine as their own commit(s) with a body listing the items, plus separate commits for UI polish and docs. Imperative summary ≤ ~70 chars, optional body, blank line, then exactly one footer: `Type:` = `feature | fix | refactor | docs | chore | test | perf` (seed data → `chore`).
6. **PLAN.md is canonical.** If a §9 example item can't be sourced from any working adapter, you may swap it for a comparable product — note the substitution in SESSION_LOG.md; a PLAN.md correction itself only gets its own `Type: docs` commit if the doc is actually wrong.
7. **Blockers:** stop if stuck; record under "Open issues" with what you tried; leave the tree coherent — never end with unexplained half-work.
8. Never commit `.venv/`, `data/`, `.env`, or any `*.db` (covered by `.gitignore`).

## Definition of done (checklist)
- [ ] "High Demand Tech" contains **≥ 8 real items** across all five categories, every row status=ok with ≥1 stored price; no `pending://` rows remain.
- [ ] At least two full scheduler cycles completed over the real list (evidence: multiple `prices`/`scrape_runs` rows per item); interval restored to defaults afterward.
- [ ] Good-buy section behaves exactly per PLAN §7 with this data: ranked deals where thresholds are met, "insufficient data" for items with <7 days of history — no faked analytics.
- [ ] Empty states + error messages render correctly; list sort dropdown works.
- [ ] README usage docs present and accurate (supported-sites table matches S6 reality).
- [ ] All commits carry `Type:` footers; SESSION_LOG updated; docker build passes (if available).

## Suggested commit order (adapt as needed)
1. "Replace placeholder seeds with verified High Demand Tech items" — Type: chore
2. "Run live scrape cycles over real seed data" — Type: chore (body: intervals used, observations)
3. "Polish dashboard empty states, error messages, list sorting" — Type: feature
4. "Document usage, supported sites, and good-buy badges in README" — Type: docs

## Finish protocol (mandatory)
1. Append a **Session 7** entry to `SESSION_LOG.md`:
   ```markdown
   ## Session 7 — <YYYY-MM-DD>
   - What was done: <bullets incl. final item list with URLs + retailers>
   - Live cycles: <intervals used, rows observed, errors seen>
   - Commits: <hash> "message" (Type: x), ...
   - Test/build status: pytest N passed; docker build ok | skipped (reason)
   - Open issues: <any items that couldn't be verified + why>
   - Notes for next session: <env/config state S8 should preserve, e.g. defaults restored>
   ```
2. Commit the log update separately — `Type: docs`.
3. Report to the user: commits made, test/build results, the final seed list (item → retailer → last price), and confirmation that **Session 8** is ready (`prompts/session-8.md`).
