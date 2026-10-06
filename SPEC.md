# Web Project Spec

A strict, single-stack template for new web projects.
Stack-agnostic in spirit; this instance commits to FastAPI + React + Postgres.
A Golang variant is deferred until a real Go project demands it.

## 1. Scope

This spec is the contract for how web apps are built and run here.
It covers project anatomy, tech stack, coding conventions, and operational concerns.
Anything not in this spec is a project decision; anything in this spec is not.

## 2. Tech Stack

| Layer | Choice |
| --- | --- |
| Backend framework | FastAPI |
| Backend language | Python 3.12 |
| Backend deps | `uv` (lockfile: `uv.lock`) |
| Backend lint/format | `ruff` (line length 88) |
| Backend logging | `structlog` (JSON to stdout) |
| Backend ASGI | `gunicorn` + `uvicorn.workers.UvicornWorker` |
| ORM | SQLAlchemy 2.0 |
| Migrations | Alembic (autogenerate + manual review) |
| Config | `pydantic-settings` |
| Database | PostgreSQL 18 |
| Frontend framework | React 18 |
| Frontend tooling | Vite |
| Frontend language | TypeScript |
| Frontend PM | `pnpm` |
| Frontend lint/format | Biome |
| Frontend styling | Tailwind CSS |
| Frontend routing | React Router v6/v7 |
| Frontend state | React Query (server) + hooks/context (client) |
| Frontend testing | Vitest + Testing Library (Playwright optional) |
| API contract | OpenAPI -> TS via `openapi-typescript` + `openapi-fetch` |
| Auth | Session cookies, Postgres-backed, httpOnly |
| Hosting | Render |
| CI/CD | GitHub Actions |
| Deploy trigger | CI-driven via Render Deploy Hook |

Runtime versions are pinned in both `.python-version` + `.nvmrc` (tool layer)
and `pyproject.toml` `requires-python` + `package.json` `engines` (manifest layer).
The two layers must agree and must match what Render uses.

## 3. Repository Topology

Monorepo. `backend/` and `frontend/` live at the repo root.

```
app-template/
  backend/
  frontend/
  .github/workflows/
  Makefile
  README.md
  AGENT.md
  SPEC.md
```

No shared package directory.
Cross-stack code sharing happens via OpenAPI codegen (see section 8), not a `shared/` folder.
If a shared types package becomes necessary later, that's a future decision.

## 4. Backend Structure

Flat feature modules.
Each feature is a self-contained package with its own router, models, schemas, service, and tests.

```
backend/
  app/
    main.py                 # assembles routers, mounts middleware, static files
    core/
      config.py             # pydantic-settings Settings class
      db.py                 # SQLAlchemy engine, session factory
      logging.py            # structlog config
      errors.py             # error envelope + exception handlers
      middleware.py         # request ID middleware
      deps.py               # shared FastAPI dependencies
    users/
      router.py             # APIRouter(prefix="/api/v1/users")
      models.py             # SQLAlchemy models
      schemas.py            # Pydantic request/response models
      service.py            # business logic
      tests/
        test_router.py      # integration tests (real DB)
        test_service.py     # unit tests (no DB)
    auth/
      ...
  alembic/
    versions/
  alembic.ini
  pyproject.toml
  Dockerfile
  .python-version
```

Conventions:

- One feature = one package under `app/`.
- A feature owns its router, models, schemas, service, and tests.
- Cross-feature imports go through service interfaces, not direct model access.
- `core/` holds cross-cutting concerns only (config, db, logging, errors, middleware, deps).
- `main.py` is thin: create app, include routers, mount middleware, mount static files.

## 5. Frontend Structure

Flat feature modules, symmetric with the backend.

```
frontend/
  src/
    features/
      users/
        UsersPage.tsx
        UserCard.tsx
        useUsers.ts         # React Query hooks
        users.test.tsx
      auth/
        ...
    shared/
      api/
        client.ts           # openapi-fetch client, configured
        schema.d.ts         # generated, gitignored
      ui/                   # UI kit primitives
      utils/
    App.tsx                 # routes, providers
    main.tsx                # entry
  index.html
  tailwind.config.ts
  biome.json
  package.json
  .nvmrc
```

Conventions:

- One feature = one folder under `features/`.
- A feature owns its pages, components, hooks, and tests.
- `shared/` holds cross-cutting code (API client, UI kit, utils) that no single feature owns.
- `shared/api/schema.d.ts` is generated, gitignored, and regenerated via `pnpm gen:api`.
- Route definitions live in `App.tsx`; feature pages are imported from `features/`.

## 6. API Style

REST, versioned in the path: `/api/v1/*`.

- Each feature router uses `APIRouter(prefix="/api/v1/<feature>")`.
- Breaking changes require a new version prefix (`/api/v2/...`) and a new router.
- Non-breaking changes (additive fields, new endpoints) stay in `v1`.
- The OpenAPI schema reflects the versioned paths; codegen types inherit them.

## 7. Error Envelope

All errors return a single consistent shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      { "field": "email", "message": "required" }
    ]
  }
}
```

- One exception handler in `core/errors.py` maps all error types (HTTP, validation, domain) to this shape.
- `code` is a machine-readable string (e.g., `AUTH_REQUIRED`, `VALIDATION_ERROR`, `NOT_FOUND`).
- `message` is human-readable, safe to show.
- `details` is optional; present for validation errors with per-field info.
- The frontend types this envelope via OpenAPI codegen and branches on `code`.

## 8. API Contract and Codegen

OpenAPI is the single source of truth.
FastAPI emits `/openapi.json` for free; the frontend consumes it.

- `openapi-typescript` generates TS types from the schema into `shared/api/schema.d.ts`.
- `openapi-fetch` provides a typed `fetch` wrapper around those types.
- Local: `pnpm gen:api` fetches `http://localhost:8000/openapi.json` (backend must be running).
- CI: frontend workflow spins up the backend (or a service container) and runs the same script.
- `schema.d.ts` is gitignored; it is always regenerated, never committed.

## 9. Database

PostgreSQL 18.
Access via SQLAlchemy 2.0 (declarative models, async or sync per project choice).

- `core/db.py` owns the engine and session factory.
- Connection string comes from `Settings.database_url` (env var).
- Pool settings are env-tuned: `pool_size`, `max_overflow`, `pool_pre_ping=True`.
- No raw SQL in feature code; use SQLAlchemy ORM or Core where appropriate.

Migrations via Alembic:

- `alembic revision --autogenerate` inspects model changes.
- The developer reviews and edits the generated migration before committing.
- Failure modes to check: column renames (autogenerate sees drop + add, not rename), data migrations (write by hand), nullable additions, default changes.
- `alembic upgrade head` is idempotent (no-op if already at head).
- In production, migrations run via Render's Pre-Deploy Command: `alembic upgrade head`.

## 10. Configuration

`pydantic-settings` loads config from environment variables.

- `core/config.py` defines a `Settings` class with typed fields.
- Required vars missing at startup = fail fast.
- Local dev uses `.env` (gitignored); `.env.example` is committed.
- Render sets env vars in the dashboard for production and preview environments.
- No layered config files, no per-env YAML; env vars are the source of truth.

## 11. Authentication

Session cookies, Postgres-backed.

- Backend issues a session ID, stores a row in a `sessions` table, sets it in an `httpOnly` cookie.
- `SameSite=Lax` (works because the stack is single-origin; FastAPI serves the frontend).
- Revocation = delete the session row. No JWT denylist complexity.
- OAuth (social login) can be layered on top later; not in the core spec.

## 12. Logging and Observability

Structured JSON logging via `structlog`.

- `core/logging.py` configures `structlog` to emit JSON to stdout.
- Per-request context (request ID, user ID) is bound via `structlog.contextvars`.
- A middleware in `core/middleware.py` generates a UUID per request, binds it as `request_id`, and sets `X-Request-ID` on the response.
- Every log line in the request chain carries the `request_id`.

Health check:

- `GET /health` returns `{"status": "ok"}` (or includes DB connectivity if desired).
- Render's deploy health check points at this endpoint.
- No auth required on `/health`.

OpenTelemetry is an upgrade path, not part of the core spec.

## 13. Testing

### Backend

Unit + integration tests, real Postgres in CI.

- Unit tests (e.g., `test_service.py`) test pure logic with no DB. Run on every save locally.
- Integration tests (e.g., `test_router.py`) test routers/repositories against a real Postgres.
- Locally: unit tests run via `make test`; integration tests run via `make test:integration` (requires `make db` running).
- CI: GitHub Actions spins up a Postgres service container and runs the full suite.
- No mocking the DB layer. Mocking Postgres gives false confidence.

### Frontend

Vitest + Testing Library for component tests.

- Component tests run in Vite's transform pipeline (fast, same config as the app).
- Mock API calls at the `openapi-fetch` client layer; do not mock React Query internals.
- Playwright is optional, recommended for critical flows (login, checkout). Not required by the spec.

## 14. Frontend State Management

- Server state: React Query (TanStack Query). Caching, refetching, invalidation handled by the library.
- Client state: React `useState`, `useReducer`, or context. No global client store mandated.
- API calls go through the typed `openapi-fetch` client; React Query wraps it.
- No Redux, no Zustand in the core spec.

## 15. Frontend Error Handling

- React Error Boundaries catch render errors and show a fallback UI.
- A top-level Error Boundary wraps the app; per-route boundaries wrap page sections.
- A global `*` route in React Router renders a not-found page.
- API errors surface via React Query's error state; the error envelope's `code` drives UI branching (e.g., `AUTH_REQUIRED` redirects to login).

## 16. Local Development

One command: `make dev`.

- Runs backend (`uv run uvicorn app.main:app --reload`) and frontend (`pnpm dev`) concurrently.
- Vite dev server runs on `:5173` and proxies `/api` to `http://localhost:8000`.
- Developer opens `http://localhost:5173` (HMR via Vite, API via proxy).

Local Postgres: `make db`.

- Runs `docker run postgres:18` on a fixed port with a dev password.
- Persists to a named volume; `make db:reset` nukes the volume for a clean slate.
- Requires Docker installed.

Other Makefile targets:

- `make install` - install backend + frontend deps.
- `make test` - run backend unit tests + frontend component tests.
- `make test:integration` - run backend integration tests (requires `make db`).
- `make lint` - run `ruff` + Biome.
- `make format` - run `ruff format` + Biome format.
- `make gen:api` - regenerate frontend OpenAPI types (requires backend running).
- `make build` - build frontend + production Docker image.

## 17. Environment Management

Two environments:

- `production` - the live app. Deployed on merge to `main` (CI-gated).
- PR previews - Render spins up a preview environment per PR with its own URL and DB.

No permanent staging environment in the core spec.
If a team needs a shared integration environment, that's an upgrade path (fork `production` config into a `staging` Render environment).

## 18. Secrets

- Production + preview secrets: set in Render dashboard as env vars.
- Local dev: `.env` (gitignored) with `.env.example` committed documenting required vars.
- GitHub Actions: encrypted secrets for the Render Deploy Hook URL and any CI-only secrets (e.g., test DB URL if needed).
- No vault, no SOPS in the core spec.

## 19. CI/CD

GitHub Actions, split workflows with path filters.

```
.github/workflows/
  backend-ci.yml      # on push to backend/** or main
  frontend-ci.yml     # on push to frontend/** or main
  deploy.yml          # on push to main, after CI passes
```

### `backend-ci.yml`

- Triggers on `backend/**` and `main`.
- Sets up Python via `.python-version`, installs deps via `uv`.
- Runs `ruff check`, `ruff format --check`.
- Spins up Postgres service container.
- Runs `alembic upgrade head` then `pytest`.

### `frontend-ci.yml`

- Triggers on `frontend/**` and `main`.
- Sets up Node via `.nvmrc`, installs deps via `pnpm`.
- Runs Biome lint + format check.
- Spins up backend service container (or uses a reusable workflow) and runs `pnpm gen:api`.
- Runs `pnpm typecheck` (tsc), `pnpm test` (Vitest), `pnpm build`.

### `deploy.yml`

- Triggers on push to `main`, after `backend-ci` and `frontend-ci` pass (via `workflow_run` or required checks).
- Calls Render Deploy Hook URL (stored in GitHub secrets) via `curl`.
- Render builds the Docker image from the repo and deploys.
- Render's Pre-Deploy Command runs `alembic upgrade head` before traffic is served.

## 20. Deployment

One Dockerfile, multi-stage, at `backend/Dockerfile`.

- Stage 1 (Node): install pnpm, build `frontend/dist`.
- Stage 2 (Python): install backend deps via `uv`, copy `frontend/dist` into the static mount path, set CMD.
- Final image has no Node runtime; only Python + built assets.
- Non-root user.
- CMD: `gunicorn app.main:app -w ${WEB_CONCURRENCY:-4} -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:${PORT:-8000}`.
- FastAPI mounts `frontend/dist` via `StaticFiles` and serves `index.html` for non-API routes (SPA fallback).

Single origin: no CORS, no cookie domain juggling, `SameSite=Lax` just works.

## 21. Frontend Serving

FastAPI serves the built frontend as static files.

- `pnpm build` outputs to `frontend/dist`.
- FastAPI mounts `StaticFiles(directory="frontend/dist")` at `/`.
- Non-API routes fall back to `index.html` (SPA routing).
- `/api/*` routes hit the FastAPI routers before the static mount.

In dev, Vite serves the frontend with HMR and proxies `/api` to FastAPI.
In prod, FastAPI serves both the API and the static files from one origin.

## 22. Git Conventions

- `main` is the default branch.
- PRs required; no direct commits to `main`.
- CI must pass before merge.
- Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`.
- Squash merges on merge to keep `main` history linear.

## 23. Documentation

- `README.md` - setup, run, test, deploy instructions. The "how to start" doc.
- `AGENT.md` - context for AI agents working on the repo: stack summary, conventions, gotchas.
- `SPEC.md` - this file. The contract.
- Per-feature docs are optional and live in the feature folder if needed.

## 24. What This Spec Does Not Decide

- Cloud services beyond Render (S3, queues, caches) - project decision.
- Feature flags - project decision.
- Internationalization - project decision.
- Email provider - project decision.
- Monitoring/alerting beyond Render's built-ins - project decision.
- Docker Compose for local dev - optional add-on, not the default.
- OpenTelemetry - upgrade path, not the default.
- OAuth/social login - can layer on top of session auth.
- A Golang variant of this spec - deferred until a real Go project requires it.
