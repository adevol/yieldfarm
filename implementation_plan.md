# Production-Lean Scaffold Plan: DeFi Pool Yield + Risk Decomposition

## Summary
Build a new monorepo scaffold with `backend` (FastAPI + sync SQLAlchemy + Alembic + APScheduler), `frontend` (React + TS + Vite + TanStack Query + React Router + ECharts), `infra` (Docker Compose + env templates), and `docs` (architecture + scoring).  
The provider layer is strictly abstracted so Dune/Flipside can be swapped without changing DB schema, metrics/scoring code, or API handlers.  
Defaults chosen: `uv` tooling, sync DB sessions, 1-second scheduler tick, env-based autopilot defaults with query overrides, ECharts.
High-frequency target: user-visible updates every 1 second (or faster) via backend live stream + frontend live rendering.

## Implementation Plan

### 1) Repository bootstrap and layout
1. Create directories exactly as requested:
`backend/src/providers/base.py`, `backend/src/providers/dune/`, `backend/src/providers/flipside/`, `backend/src/domain/`, `backend/src/api/`, `backend/src/ingest/`, `backend/tests/`, `backend/alembic/`, `frontend/src/`, `infra/`, `docs/`.
2. Add root `README.md` with quickstart and architecture links.
3. Add `.gitignore` for Python, Node, Vite, pytest, mypy, Alembic artifacts, and local env files.

### 2) Backend project setup
1. Use `backend/pyproject.toml` with `uv` workflow and dependencies: `fastapi`, `uvicorn`, `pydantic-settings`, `sqlalchemy`, `psycopg2-binary`, `alembic`, `apscheduler`, `httpx`, `python-dotenv`.
2. Add dev dependencies: `ruff`, `mypy`, `pytest`, `pytest-cov`, `types-requests` (if needed), plus FastAPI typing helpers.
3. Add `backend/src/main.py` app factory with CORS enabled for local frontend origin(s).
4. Add config module (`settings.py`) reading env:
`DATA_PROVIDER`, `DUNE_API_KEY`, Flipside stub vars, `DATABASE_URL`, `ADMIN_API_KEY`, scheduler tick interval, provider min-fetch interval, autopilot defaults, pool config JSON.
5. Add SQLAlchemy sync engine/session setup and FastAPI dependency injection for sessions.
6. Add health router and API router mounting under `/api`.

### 3) Database schema and migrations
1. Add ORM models:
`Pool`, `PoolSnapshot`, `RawIngest`.
2. Add Alembic initial migration for required canonical tables.
3. Add idempotency constraints/indexes:
`pools`: unique on `(protocol, chain, pool_address)`.
`pool_snapshots`: unique on `(pool_id, ts, source)` to prevent duplicate writes.
`raw_ingest`: unique on `(provider, key, checksum)` for safe retry dedupe.
4. Use `jsonb` for `metadata` and `payload`; add useful indexes on timestamps and provider/key.

### 4) Provider abstraction and implementations
1. Define provider interface in `providers/base.py`:
`name`, `fetch_raw(targets, start, end)`, `normalize(raw_payloads) -> canonical snapshot records`.
2. Implement `providers/dune/client.py` real provider:
reads `DUNE_API_KEY`, executes configured query requests, returns raw payload envelopes with stable ingest keys.
3. Implement `providers/flipside/client.py` stub:
same interface, placeholder methods, explicit TODO markers for credentials and query execution.
4. Implement provider factory:
`DATA_PROVIDER=dune|flipside`; fallback to mock provider mode when Dune key absent.
5. Implement mock provider with seed dataset loaded from `backend/src/providers/mock/seed_data.json`.

### 5) Ingestion runner and scheduler
1. Create `ingest/runner.py` with transaction-safe ingestion flow:
fetch raw -> write `raw_ingest` deduped -> normalize -> upsert pools -> insert snapshots with conflict handling.
2. Create `ingest/scheduler.py` using APScheduler interval trigger (default 1s tick from env).
3. Add provider-safe throttling in scheduler state (`last_fetch_at` per target/provider) so the scheduler ticks every second but only calls external providers when `PROVIDER_MIN_FETCH_SECONDS` is satisfied.
4. In mock mode, allow true 1s synthetic snapshot generation from seed baselines for real-time UX/dev.
5. Create `POST /api/admin/ingest` manual trigger endpoint; protect with `X-Admin-Key` header matched to `ADMIN_API_KEY`.
6. Ensure retries are safe via unique constraints + conflict handling; never duplicate snapshots for same `(pool_id, ts, source)`.

### 6) Domain metrics, scoring, and simulation
1. Add provider-agnostic domain schemas in `domain/schemas.py` (Pydantic response models).
2. Add `domain/metrics.py`:
rolling volatility on `supply_apy` using window `N` (env default).
utilization delta and spike flag via threshold (env default).
3. Add `domain/scoring.py` with explicit formulas:
`stability_score = clamp(100 - a*volatility - b*util_spike_penalty, 0, 100)`.
`risk_adjusted_score = w1*supply_apy + w2*stability_score + w3*incentive_apy - w4*borrow_apy`.
4. Add `domain/simulation.py` autopilot allocator:
filter pools by `min_tvl_usd`,
rank by `risk_adjusted_score`,
allocate up to `max_weight_per_pool`,
apply cooldown rebalance suppression using pool-level last-allocation timestamps in simulation state.
5. Allow constraint overrides from query params; fallback to env defaults.

### 7) FastAPI endpoints
1. `GET /healthz`: DB connectivity + scheduler status lightweight response.
2. `GET /api/pools`: leaderboard list with latest snapshot + computed scores, sortable server-side.
3. `GET /api/pools/{pool_id}`: pool metadata, latest metrics, and timeseries arrays.
4. `GET /api/simulations/autopilot?start=&end=&max_weight=&min_tvl=&cooldown_hours=`: run backtest-style simulation over snapshots.
5. `POST /api/admin/ingest`: authenticated manual ingestion trigger.
6. `GET /api/stream/pools`: Server-Sent Events (SSE) stream emitting leaderboard snapshot updates every second.
7. Standardize error model and validation errors via FastAPI defaults + custom `detail` for admin auth failures.

### 8) Frontend app scaffold
1. Create Vite React TS app in `frontend/`.
2. Add dependencies: `@tanstack/react-query`, `react-router-dom`, `echarts`, `echarts-for-react` (or equivalent wrapper), and a lightweight table utility.
3. Routing:
`/` Pools table page,
`/pools/:poolId` detail page,
`/simulations/autopilot` simulation page.
4. Build API client with typed fetch helpers and shared error handling.
5. Implement charts with ECharts:
Pool detail chart 1: APY timeseries line chart.
Pool detail chart 2: utilization and/or TVL chart (dual axis if useful).
Simulation page chart: allocation or portfolio APY over time.
6. Add sortable leaderboard table and navigation links.
7. Add React Query provider, caching keys, and loading/error states.
8. Add live update adapter:
connect to `/api/stream/pools` via `EventSource` for 1s pushes,
fallback to TanStack Query polling (`refetchInterval: 1000`) if SSE is unavailable.

### 9) Infra and runtime
1. `infra/docker-compose.yml` services:
`db` Postgres with volume + healthcheck,
`backend` FastAPI with reload for local dev,
`frontend` Vite dev server.
2. `infra/env.example` with all backend/frontend vars including provider credentials, pool config JSON, scoring weights, scheduler interval, and admin key.
3. Backend Dockerfile with uv install path and app startup command.
4. Optional frontend Dockerfile for containerized dev in compose (included for consistency).
5. Compose command path in README: `docker compose -f infra/docker-compose.yml --env-file infra/.env up --build`.

### 10) Tooling, quality gates, and docs
1. Backend lint/type/test commands in `pyproject.toml` scripts:
`ruff check`, `ruff format`, `mypy`, `pytest`.
2. Frontend ESLint + Prettier configs and npm scripts:
`lint`, `format`, `typecheck`, `test` (placeholder if no test runner yet).
3. Add baseline backend tests:
provider factory selection,
mock ingestion inserts expected rows,
idempotent re-run produces no duplicates,
metrics/scoring deterministic outputs,
admin endpoint auth success/failure.
4. Add frontend baseline tests optional; if not included, document as follow-up.
5. `docs/architecture.md` covering provider abstraction contract, ingestion flow, schema rationale, and scoring/simulation design.
6. README includes:
setup, env, running compose, mock mode behavior, adding pools to config, endpoint list, and scoring formula explanation.

## Public APIs / Interfaces / Types (Important Additions)
1. Provider protocol (`backend/src/providers/base.py`):
`fetch_raw(targets, start, end)`, `normalize(raw_payloads)`; canonical outputs independent of source provider.
2. Canonical DB schema:
`pools`, `pool_snapshots`, `raw_ingest` with provider-agnostic snapshot fields.
3. API contracts:
`GET /api/pools`, `GET /api/pools/{pool_id}`, `GET /api/simulations/autopilot`, `POST /api/admin/ingest`, `GET /api/stream/pools`, `GET /healthz`.
4. Config interfaces:
`DATA_PROVIDER`, provider creds, `SCHEDULER_TICK_SECONDS`, `PROVIDER_MIN_FETCH_SECONDS`, scheduler/scoring/autopilot defaults, `POOLS_CONFIG_JSON`.
5. Frontend typed models aligned to backend response schemas for leaderboard, detail timeseries, and simulation outputs.

## Test Cases and Scenarios
1. Ingestion with `DATA_PROVIDER=dune` and no `DUNE_API_KEY` uses mock provider and seeds successfully.
2. Ingestion with valid Dune key path calls Dune provider and persists raw + normalized rows.
3. Re-running ingest on same payload does not duplicate `raw_ingest` or `pool_snapshots`.
4. Pool leaderboard returns sorted pools and includes computed `stability_score` and `risk_adjusted_score`.
5. Pool detail returns full timeseries and handles unknown pool ID with 404.
6. Autopilot simulation respects `min_tvl`, `max_weight_per_pool`, and `cooldown_hours`.
7. Admin ingest endpoint rejects missing/invalid `X-Admin-Key` and accepts valid key.
8. SSE stream emits updates at 1s cadence and frontend reflects updates within 1s.
9. Scheduler ticks every 1s but external provider calls honor `PROVIDER_MIN_FETCH_SECONDS`.
10. Docker compose boots all services; frontend can query backend from configured base URL.

## Assumptions and Defaults
1. Time handling is UTC end-to-end.
2. Scheduler tick default is every 1s; real provider fetch cadence is rate-limit-aware via `PROVIDER_MIN_FETCH_SECONDS`.
3. `X-Admin-Key` is the auth header for admin ingest.
4. Autopilot defaults come from env and can be overridden per request.
5. Flipside implementation is scaffolded/stubbed but not production-integrated in MVP.
6. Mock seed dataset is included and enabled automatically when Dune credentials are absent.
7. Sync SQLAlchemy is used consistently across app and tests for MVP simplicity.
