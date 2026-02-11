# Roadmap: DeFi Pool Yield + Risk Decomposition

## Phase 0: Repo Skeleton
Goal: create the monorepo structure and base docs.

Tasks:
1. Create `backend/`, `frontend/`, `infra/`, `docs/`.
2. Add requested backend folders (`providers`, `domain`, `api`, `ingest`, `tests`, `alembic`).
3. Add root `.gitignore` and initial `README.md`.

Exit criteria:
1. Folder structure matches `implementation_plan.md`.
2. Repo is ready for backend/frontend initialization.

## Phase 1: Backend Core Setup
Goal: boot a runnable FastAPI app with config and DB session plumbing.

Tasks:
1. Create `backend/pyproject.toml` using `uv`.
2. Add FastAPI app entrypoint, CORS, and router mounting.
3. Add settings module for env vars (`DATA_PROVIDER`, DB, admin key, scheduler, scoring).
4. Add SQLAlchemy sync engine/session dependency.

Exit criteria:
1. Backend starts locally.
2. `GET /healthz` route exists (stub allowed before DB checks).

## Phase 2: Canonical DB Schema + Alembic
Goal: establish provider-agnostic persistence model.

Tasks:
1. Implement ORM models: `Pool`, `PoolSnapshot`, `RawIngest`.
2. Add initial Alembic migration for canonical tables.
3. Add unique constraints and indexes for idempotency and query performance.

Exit criteria:
1. Migration applies cleanly on Postgres.
2. Required constraints exist:
`(protocol, chain, pool_address)`, `(pool_id, ts, source)`, `(provider, key, checksum)`.

## Phase 3: Provider Abstraction Layer
Goal: isolate external data providers behind one contract.

Tasks:
1. Define provider interface/protocol in `backend/src/providers/base.py`.
2. Implement Dune provider (real integration path).
3. Implement Flipside provider stub with TODO placeholders.
4. Implement provider factory using `DATA_PROVIDER`.
5. Add mock provider + seed dataset for no-key local mode.

Exit criteria:
1. Backend can run with `DATA_PROVIDER=dune` and missing key by falling back to mock mode.
2. Provider swap does not require DB/API/domain changes.

## Phase 4: Ingestion Engine + Scheduler
Goal: build idempotent ingestion with fast update cadence.

Tasks:
1. Implement ingestion runner:
fetch raw -> persist `raw_ingest` -> normalize -> upsert pools -> insert snapshots.
2. Implement APScheduler 1-second tick.
3. Add provider-safe throttling via `PROVIDER_MIN_FETCH_SECONDS`.
4. Add manual admin ingest trigger endpoint auth (`X-Admin-Key`).

Exit criteria:
1. Scheduler runs every second.
2. Re-ingesting same payload does not duplicate canonical rows.
3. External calls respect provider min-fetch interval.

## Phase 5: Domain Metrics + Scoring + Autopilot
Goal: provider-agnostic analytics and allocation logic.

Tasks:
1. Add domain schemas for leaderboard/detail/simulation outputs.
2. Implement rolling APY volatility and utilization spike detection.
3. Implement `stability_score` and `risk_adjusted_score`.
4. Implement autopilot allocation with constraints:
max pool weight, min TVL, cooldown.

Exit criteria:
1. Scores are deterministic for fixed input snapshots.
2. Simulation honors all constraints and query overrides.

## Phase 6: API Surface
Goal: expose full MVP backend contract.

Tasks:
1. Implement `GET /healthz`.
2. Implement `GET /api/pools` leaderboard endpoint.
3. Implement `GET /api/pools/{pool_id}` detail + timeseries.
4. Implement `GET /api/simulations/autopilot`.
5. Implement `POST /api/admin/ingest` with API key protection.
6. Implement `GET /api/stream/pools` SSE for 1s live updates.

Exit criteria:
1. All endpoints return typed, validated responses.
2. SSE stream emits updates at 1s cadence.

## Phase 7: Frontend Foundation
Goal: create the frontend shell and data plumbing.

Tasks:
1. Initialize React + TypeScript + Vite app.
2. Add React Router and TanStack Query providers.
3. Add typed API client and route structure:
`/`, `/pools/:poolId`, `/simulations/autopilot`.

Exit criteria:
1. Frontend app boots and navigates across all pages.
2. API client can fetch backend data in dev setup.

## Phase 8: Frontend Features + Live UX
Goal: deliver MVP UI functionality and charts.

Tasks:
1. Build sortable pools leaderboard table.
2. Build pool detail page with at least two ECharts charts.
3. Build autopilot simulation page with charted results.
4. Add live updates using SSE `EventSource`.
5. Add fallback 1s polling with TanStack Query if SSE fails.

Exit criteria:
1. User-visible data refreshes every 1 second (or faster in SSE mode).
2. Required charts are rendered and bound to real API responses.

## Phase 9: Infra + Local Runtime
Goal: one-command local startup with Docker Compose.

Tasks:
1. Create `infra/docker-compose.yml` for Postgres, backend, frontend.
2. Add backend Dockerfile and optional frontend Dockerfile.
3. Add `infra/env.example` with all required vars.

Exit criteria:
1. `docker compose up` starts all services.
2. Frontend can reach backend, backend can reach Postgres.

## Phase 10: Quality Gates + Documentation
Goal: make scaffold production-lean and maintainable.

Tasks:
1. Add backend lint/type/test tooling (`ruff`, `mypy`, `pytest`).
2. Add frontend lint/format tooling (ESLint + Prettier).
3. Add backend baseline tests for provider selection, ingestion idempotency, metrics, admin auth, and 1s stream/scheduler behavior.
4. Write `docs/architecture.md`.
5. Expand `README.md` with setup, envs, scoring formula, adding pools, and mock mode.

Exit criteria:
1. Tooling commands run successfully.
2. Docs cover architecture and operational workflow end-to-end.

## Suggested Execution Order
1. Phase 0 -> 1 -> 2 -> 3 -> 4
2. Phase 5 -> 6
3. Phase 7 -> 8
4. Phase 9 -> 10
