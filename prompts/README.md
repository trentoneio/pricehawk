# Session Prompts — PriceHawk

One prompt file per build session (`session-0.md` … `session-8.md`). Each one is **fully self-contained**: it assumes zero memory of any previous session, relying only on the state of this repository (code, git history, `SESSION_LOG.md`) and this directory.

## How to run a session
1. Start with the next unstarted session: read that file in full from the top (`prompts/session-N.md`).
2. The prompt's "Start here" checklist tells you exactly what prior state to verify before writing anything — if it doesn't match, stop and report instead of guessing.
3. Follow the prompt's deliverables, hard rules, definition-of-done checklist, and finish protocol (which includes updating `SESSION_LOG.md` and reporting commits).

## Session index
| File | Scope |
|---|---|
| `session-0.md` | Repo scaffolding: FastAPI skeleton, pinned deps, tests, Dockerfile/compose skeletons, README, SESSION_LOG |
| `session-1.md` | Data layer: SQLAlchemy models, DB init (WAL), repository functions, seed CLI with "High Demand Tech" placeholders |
| `session-2.md` | Scraper core (base contracts, shared rate-limited http client, JSON-LD helper) + Newegg adapter + fixtures/tests |
| `session-3.md` | Background scheduler: APScheduler ticker job, backoff, list gating, live auto-scrape proof |
| `session-4.md` | Dashboard UI + lists management (add-item box, tracked/untracked toggles, sparklines) |
| `session-5.md` | Vendored Chart.js price charts + good-buy scoring engine per PLAN §7 |
| `session-6.md` | Remaining adapters: Micro Center, Best Buy (+ optional Playwright), Amazon (best-effort/stub), Server Parts Deals |
| `session-7.md` | Replace placeholder seeds with real verified items, live scrape cycles, UI polish, usage docs |
| `session-8.md` | Final Dockerfile (non-root + healthcheck), `.env.example`, Dockage-ready compose, TrueNAS deployment guide, clean-room verification |

## Ground rules baked into every prompt
- Do only that session's scope; never start later sessions' work.
- End with: tests green, docker build passing (if Docker is available locally), a coherent tree, and a `SESSION_LOG.md` entry — all committed with the repo's commit convention (imperative summary + body + single trailing `Type:` footer).
