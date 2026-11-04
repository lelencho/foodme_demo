# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

FoodMe is a food-ordering demo used for a QA/observability course. Three apps live under `apps/`, plus a monitoring stack under `infra/monitoring/`:

- `apps/backend`: Spring Boot 3.3 (Java 17, Gradle), PostgreSQL + Flyway, JWT auth.
- `apps/web`: customer storefront. React 19 + TypeScript + Vite 8 + Tailwind 4 + TanStack Query, with Radix/shadcn-style UI in `src/components/ui`.
- `apps/admin`: back office. react-admin 5 + MUI, React 18, plain JS/JSX.

Some behavior is intentionally fake, for the course. Don't "fix" it unless asked:
- `SimulatedLatencyConfig` adds a random 200–1500 ms delay to `/api/**` and `/admin/**`. `/api/images/**` is excluded, and the delay is 0 under the `test` profile.
- `FlakyHeartbeatJob` fails about 1 run in 10 and reports to Sentry/GlitchTip. It's disabled under `test`.
- `GET /api/debug/boom` always throws, so the error tracker has something to catch.
- The `flakyHeartbeat` files in both frontends, and the `flake-*.spec.ts` e2e specs, are deliberate flakiness.
- `01-introduce-bug-FM-BUG-07 (1).patch` seeds a bug (cart decrement threshold in `apps/web/src/hooks/useCart.ts`) for reproduce/fix exercises tracked in Jira. The fix is already merged on `main`.

## Commands

Run each app from its own directory. There's no root package.json or workspace tool.

**Backend** (`apps/backend`):
```bash
./gradlew bootRun                                   # needs Postgres on localhost:5432, db/user/pass "foodme"
./gradlew test                                      # H2 in-memory, profile "test", no Postgres needed
./gradlew test --tests am.foodme.backend.DishControllerTest
./gradlew test --tests 'am.foodme.backend.DishControllerTest.someMethod'
./gradlew bootJar
```
Swagger UI is at `/swagger-ui.html` and health at `/actuator/health`. The default port is 8081 (`PORT` overrides it).

**Web** (`apps/web`) and **Admin** (`apps/admin`):
```bash
npm ci
npm run dev        # dev servers call the backend at http://localhost:8081
npm run build      # web: tsc -b && vite build (type errors fail the build)
npm run lint       # web: oxlint, admin: eslint
npm run test:e2e   # Playwright; starts its own Vite dev server
npx playwright test e2e/happy-path.spec.ts          # single spec
npx playwright test -g "test name"                  # single test by title
```
Web e2e runs on port 5180 (`PLAYWRIGHT_BASE_URL` overrides it). Admin e2e runs on 5174 (`ADMIN_BASE_URL`), serially. Both expect a backend running on :8081 with seeded data. `npm run test:e2e:all` in `apps/web` runs both suites.

**Full production image** (repo root as the build context):
```bash
docker build -f apps/backend/Dockerfile .
```

`.env.example` mentions `infra/docker-compose.yml` and Jenkins/GlitchTip compose profiles. Those files aren't in this repo, so don't rely on them.

## Architecture

**One origin in production.** `apps/backend/Dockerfile` builds both SPAs and copies them into the backend jar: web into `static/`, admin into `static/backoffice/`. `config/SpaWebConfig` serves them:
- The storefront is at `/` and the admin SPA is at `/backoffice`. Admin is built with Vite `base: '/backoffice/'` in production mode only.
- `/admin/**` is the **admin REST API**, not the admin UI. Keep that distinction when adding routes.
- Paths without a file extension fall back to the matching `index.html`, except `api/`, `admin/`, `actuator/`, `swagger-ui`, and `v3/`. A new top-level backend prefix must be added to that exclusion list.
- Both frontends use an empty API base in production (relative calls) and `http://localhost:8081` in dev. An explicit `VITE_API_BASE_URL` overrides both. See `apps/web/src/api/client.ts` and `apps/admin/src/api/base-api.js`.

**Backend layout** (`am.foodme.backend`): `controller/api` holds public and customer endpoints, and `controller/admin` holds admin endpoints. They call `service`, which uses Spring Data `repository` (custom dish search lives in `DishSearchRepositoryImpl`). The code also has `dto` and JPA `model` packages. Route prefixes are constants in `utils/ControllerUtil`. Errors go through `exceptionHandler/GlobalExceptionHandler` (throw `NotFoundException` / `BadRequestException`).

**Auth.** Auth is stateless JWT (`security/JwtService`, `JwtAuthenticationFilter`, `SecurityConfig`). Customers and admins are separate entities with separate login endpoints (`/api/auth/**`, `/admin/auth/login`). Rules in `SecurityConfig`:
- `/api/customer/**` and `POST /api/order` require `ROLE_CUSTOMER`.
- The rest of `/api/**` is public.
- `GET /admin/dish/**` is public. Everything else under `/admin/**` requires authentication.

The web app stores its token through `lib/auth-storage.ts`. Admin keeps it in `localStorage.token` and redirects to `login` on a 401.

**Database.** The schema `foodme` is managed by Flyway (`src/main/resources/db/migration/V*.sql`), and Hibernate runs with `ddl-auto=validate`. Schema changes therefore need a new `V{n}__*.sql` migration, and the entities have to match it. Tests skip Flyway: they use H2 in PostgreSQL mode with `create-drop` plus `src/test/resources/data.sql`. Entity changes may also need that seed file updated.

Datasource resolution, in order:
1. `SPRING_DATASOURCE_URL`.
2. `DATABASE_URL`. `config/DatabaseUrlEnvironmentPostProcessor` (registered in `META-INF/spring.factories`) converts Render/Neon `postgresql://user:pass@host/db` URIs to JDBC.
3. The separate `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` variables.

**Images** are stored in Postgres and served from `/api/images/**`. On first start, `ImageSeedRunner` loads `src/main/resources/img-seed/` into the DB (`foodme.images.seed-enabled`). `ImageUrlResponseAdvice` rewrites image URLs in responses.

**Storefront state.** The cart lives client-side in IndexedDB via Dexie (`src/lib/db.ts`, `src/hooks/useCart.ts`) and is not stored on the server. Server data goes through TanStack Query, with calls in `src/api/foodme.ts`. Forms use react-hook-form + zod (`src/schemas`), i18n uses i18next (`src/locales/en/translation.json`), and the path alias `@` maps to `src`.

**Observability.**
- The backend exposes `/actuator/prometheus` and writes JSON logs (`logback-spring.xml`).
- Logs are pushed to Loki only when `LOKI_PUSH_URL` is set.
- Sentry goes to GlitchTip through `SENTRY_DSN` for the backend and `VITE_SENTRY_DSN` for the frontends. The frontend DSN is baked in at build time.
- `observability/HttpLoggingFilter` logs `/api/**` and `/admin/**` requests with secrets redacted. Turn it off with `HTTP_LOGGING_ENABLED=false`.

## Deployment

The deployment target is Render's free tier.
- `render.yaml` deploys the app (the Docker image above) plus a free Postgres, with `autoDeploy: true`.
- `render-monitoring.yaml` deploys `infra/monitoring/stack`: one container running Prometheus, Loki, Grafana, and Grafana MCP behind nginx at `/`, `/prom/`, `/loki/`, and `/mcp`. Its `autoDeploy` is off on purpose, because every redeploy wipes its data.
- The services are connected by hand: set `BACKEND_HOST` on monitoring and `LOKI_PUSH_URL` on the app. Free Render services can't receive private-network traffic, so these must be public `*.onrender.com` URLs.

The JVM flags in the backend Dockerfile and `server.tomcat.threads.max=32` are tuned for the 512 MB / 0.1 CPU free instance, so startup fits Render's health-check window. Keep them unless you're changing the hosting plan.
