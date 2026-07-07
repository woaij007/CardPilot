# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Current state

This repo is **pre-scaffold**. The only substantive content is the product requirements document at `doc/proposal.md` — read it first; it is the source of truth for scope, architecture, and phasing. Code (`frontend/`, `backend/`, `docker-compose.yml`) does not exist yet and is created in milestone **M0** (see the roadmap in `doc/proposal.md` §10). The commands below describe the *intended* toolchain per the proposal; treat them as the target once scaffolding lands, and update this file when the actual scripts exist.

## What CardPilot is

A US-market web app (responsive web + PWA) that answers "**which of my cards should I use for this purchase?**" The user picks a friendly spending category; the app ranks the cards they already hold by effective cash reward. See `doc/proposal.md` §1.

## Architecture decisions that shape the whole codebase

These are non-obvious and load-bearing — get them wrong and the design falls apart:

- **The recommendation engine runs entirely client-side, in TypeScript.** The backend has no recommendation logic. The browser caches the card catalog and computes the ranking locally over the wallet. (`doc/proposal.md` §6, §8.1)
- **No login and no server-side user data in MVP.** The wallet (held cards, cap usage, activation state, cpp overrides) lives only in browser `localStorage`. It is *never sent to the server*. Accounts + Google OAuth + cloud sync are Phase 2. (§4.1, §4.2, §7 Privacy)
- **The backend's entire job is: serve/version the card catalog + host admin editing.** Nothing else. (§8.1)
- **Catalog delivery is a versioned conditional full-fetch.** `GET /catalog?since=<version>` returns the *whole* bundle (cards, rules, rotations, currencies) if newer than `<version>`, else `304`. It is not a delta API. The PWA caches the bundle and re-fetches only when the version advances. (§8.2)
- **Rewards are normalized to dollars before ranking.** Every rewards currency has an editorial cents-per-point (cpp) value; points-vs-cashback comparisons always happen in dollars, never raw points/multipliers. Users can override cpp locally. (§5.3, §6)
- **Reward rules are stored relationally, not as JSON blobs**, so they stay queryable and versionable (rules are effective-dated). (§8.1, §4.5 ADM-2)
- **Friendly categories ≠ raw MCCs.** MVP does *not* build an MCC-resolution engine. Instead each `RewardRule` carries `merchant_exceptions` (included/excluded merchants + note) surfaced as caveats (e.g. "Walmart/Target excluded from Groceries"); the user judges applicability. Automatic MCC resolution is Phase 2. (§5.1, §4.3 REC-6, §5.2)

## Scope discipline

MVP is deliberately narrow and is for **internal testing / dogfooding first**, not a public launch — there is no KPI framework or usage analytics. Before adding anything, check it isn't explicitly deferred: new-card suggestions, accounts/login, MCC engine, travel optimizer, transaction sync, native app, and affiliate links are all **out of MVP** (Phase 2+). See `doc/proposal.md` §1.3 and §3.2.

## Intended stack & layout (target for M0)

- **Frontend**: React + Vite + TypeScript, SPA + PWA (vite-plugin-pwa). Owns the recommendation engine as a pure TS module. TanStack Query for the catalog fetch; React Router.
- **Backend**: Python + FastAPI (Pydantic v2), SQLAlchemy 2.x + PostgreSQL, Alembic migrations. Admin via SQLAdmin or minimal internal pages.
- **Layout** (`doc/proposal.md` §8.3): `frontend/`, `backend/`, `docker-compose.yml` (api + postgres + frontend), `doc/`.

## Commands (to be created in M0 — not yet runnable)

Once scaffolded, expect roughly:

```bash
# Full local dev (api + postgres + frontend)
docker-compose up

# Frontend (in frontend/)
npm run dev            # Vite dev server
npm run build          # production build
npm run lint
npm test               # unit tests (recommendation engine is the priority target)

# Backend (in backend/)
uvicorn app.main:app --reload   # dev server, OpenAPI at /docs
alembic upgrade head            # apply migrations
pytest                          # run tests
pytest path::test_name          # run a single test
ruff check . && mypy .          # lint + typecheck
```

CI (GitHub Actions) runs lint, typecheck, and tests on PRs (§8.1). Update this section with the real scripts once `package.json` / `pyproject.toml` exist.

## Git / commit discipline

- **One logical change per commit.** After making an update, commit *that* change on its own before starting the next one. Do not batch unrelated edits into a single commit.
- If a request touches several distinct concerns, split them into separate commits, each with a focused message describing only that change.
- Commit and push are separate steps: only push when the user asks. Committing does not imply pushing.

## Conventions

- **`PROMPT.md` is gitignored** — do not commit it or reference it in tracked files.
- The recommendation logic is the highest-value code to test: it is a pure function over (purchase, wallet, catalog); keep it pure and deterministic so it is exhaustively unit-testable (§4.3 REC-7).
- Every reward-value figure shown to the user is an estimate — the UI must carry the "informational only, not financial advice" disclaimer (§7 Disclaimers).
