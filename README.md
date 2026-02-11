# Yieldfarm

Yieldfarm helps users compare DeFi lending pools in plain terms: not just "highest APY," but "best yield for the amount of risk taken."

Most dashboards show raw numbers and leave the risk judgment to the user. This project solves that by:

- ingesting pool data from interchangeable providers (Dune now, Flipside later)
- normalizing it into one canonical schema
- computing stability and risk-adjusted scoring
- simulating an "autopilot" allocation strategy with practical constraints

The goal is a production-lean full-stack app that updates quickly and can run locally with Docker.

## Repository Layout

- `backend/`
  FastAPI service, provider abstraction, ingestion/scheduler, domain scoring logic, tests, and Alembic migrations.
- `frontend/`
  React + TypeScript + Vite client with routing, query caching, tables, and charts.
- `infra/`
  Docker Compose and env templates for local development.
- `docs/`
  Architecture and implementation notes.
- `roadmap.md`
  Phase-by-phase execution plan.
- `implementation_plan.md`
  Detailed technical plan and API/schema decisions.

## Install and Run

### Prerequisites

- Docker Desktop (or Docker Engine + Compose plugin)
- Git

### Quick Start (Docker, recommended)

1. Clone the repository and open it:
   ```bash
   git clone <your-repo-url>
   cd yieldfarm
   ```
2. Create local env file from the example once available:
   Unix/macOS:
   ```bash
   cp infra/env.example infra/.env
   ```
   Windows PowerShell:
   ```powershell
   Copy-Item infra/env.example infra/.env
   ```
3. Start the stack:
   ```bash
   docker compose -f infra/docker-compose.yml --env-file infra/.env up --build
   ```
4. Stop the stack:
   ```bash
   docker compose -f infra/docker-compose.yml down
   ```

### Local Development (without Docker)

Backend (planned stack):
1. Install Python 3.11+ and `uv`.
2. Install dependencies from `backend/pyproject.toml`.
3. Run API with reload (FastAPI + Uvicorn).

Frontend (planned stack):
1. Install Node.js 20+.
2. Install dependencies in `frontend/`.
3. Run the Vite dev server.

## Notes

- The scaffold is being built in phases defined in `roadmap.md`.
- Provider abstraction is a hard requirement: switching data source must not require DB schema or API handler changes.
