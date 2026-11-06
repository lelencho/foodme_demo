# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

FoodMe is a food-ordering demo used as a **QA / observability course project**. It's a monorepo:

| App | Stack | Dir |
|---|---|---|
| Backend | Spring Boot 3.3, Java 17, Gradle, JPA + Flyway, Postgres | `apps/backend` |
| Storefront | React 19 + TS, Vite, Tailwind 4 | `apps/web` |
| Admin back office | react-admin 5 + MUI, plain JS | `apps/admin` |
| Monitoring | Grafana + Prometheus + Loki + Grafana MCP in one service | `infra/monitoring` |

Detailed, path-scoped guidance lives in `.claude/rules/`: `backend.md`, `db.md`, `frontend-web.md`, `frontend-admin.md`, `testing-backend.md` and `testing-frontend.md`.

## Intentional demo behaviour (don't "fix" unless asked)
- Simulated API latency of 200–1500 ms on `/api/**` and `/admin/**` (`SimulatedLatencyConfig`).
- "Flaky heartbeat" jobs in the backend, web and admin that send fake errors to GlitchTip/Sentry.
- `e2e/flake-*.spec.ts`: deliberately flaky tests.
- `01-introduce-bug-FM-BUG-07 (1).patch`: a seeded bug in the web cart (`apps/web/src/hooks/useCart.ts`) for reproduce-and-fix exercises tracked in Jira.

## Single-origin architecture
`apps/backend/Dockerfile` (build context = repo root) builds both SPAs and bundles them into the backend jar:
- Storefront at `/`.
- Admin at `/backoffice` (because `/admin/**` is the admin REST API).
- REST API at `/api/**` and `/admin/**`.
- Health at `/actuator/health`, Swagger at `/swagger-ui.html`.

In dev, each app runs separately: backend on :8081, and Vite dev servers whose frontends call `http://localhost:8081`.

## Deployment
The target is Render's free tier:
- `render.yaml`: the app plus a Postgres database. Change its service `name` per fork.
- `render-monitoring.yaml`: the monitoring stack.

They're linked by two env vars: `BACKEND_HOST` on the monitoring service and `LOKI_PUSH_URL` on the app. See `README.md` and `infra/monitoring/README.md`. `.env.example` mentions `infra/docker-compose.yml` and Jenkins, but neither exists in the repo.
