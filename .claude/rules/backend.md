---
paths:
  - "apps/backend/**"
---

# Backend (Spring Boot)

Spring Boot 3.3, Java 17, Gradle, Lombok. Run from `apps/backend`:
- `./gradlew bootRun`: runs on port 8081 (or `$PORT`). Needs Postgres; see `db.md`.
- `./gradlew bootJar`

## Layout (`am.foodme.backend`)
- `controller/api/*`: public and customer API under `/api/**`.
- `controller/admin/*`: admin API under `/admin/**`. Admin list endpoints return `{ list, count }` (`AdminListResponseDto`), which the admin SPA relies on.
- Request flow: controller → `service/*` → `repository/*` → `model/*`. Wire types live in `dto/*`, and controllers return DTOs, not entities.
- `config/`, `observability/`, `security/`: cross-cutting code (described below).

## Security
`security/SecurityConfig` uses stateless JWT (`JwtService`, `JwtAuthenticationFilter`, auth0 `java-jwt`):
- `/api/auth/**`: public.
- `/api/customer/**` and `POST /api/order`: `ROLE_CUSTOMER`.
- The rest of `/api/**`, plus `/actuator/**` and Swagger: public.
- `/admin/auth/login` and `GET /admin/dish/**`: public.
- The rest of `/admin/**`: authenticated.

When you add an endpoint, check which of these rules covers it.

## Routing and static files
`SpaWebConfig` serves the bundled SPAs: the storefront at `/` and the admin at `/backoffice`. Any request without a file extension falls back to the SPA's `index.html`, except for paths under `api/`, `admin/`, `actuator/`, `swagger-ui` and `v3/`. If you add a new top-level server path, add it to that exclusion list. Never mount anything at `/admin` for the UI.

## Cross-cutting behaviour
- `SimulatedLatencyConfig`: delays `/api/**` and `/admin/**` by `foodme.latency.min-ms`–`max-ms` (default 200–1500 ms). `/api/images/**` is excluded. This is intentional.
- `observability/FlakyHeartbeatJob`: sends deliberate fake errors to Sentry/GlitchTip. It is disabled under the `test` profile.
- `observability/HttpLoggingFilter`: logs `/api/**` and `/admin/**` requests with headers and bodies, secrets redacted. Toggle with `HTTP_LOGGING_ENABLED`. Redact any new secret-bearing field.
- Logging: JSON logs (logstash encoder). They are pushed to Loki only when `LOKI_PUSH_URL` is set, via an `<if>` in `logback-spring.xml` that needs janino. Errors go to Sentry when `SENTRY_DSN` is set.
- Metrics: actuator exposes `health`, `info` and `prometheus`, with an HTTP latency histogram for p95/p99. Grafana dashboards in `infra/monitoring` depend on these metric names and the `app="foodme-backend"` label.
- `ImageUrlResponseAdvice` rewrites image URLs in responses.

## Runtime constraints
The Render free tier gives 512 MB RAM and 0.1 CPU. Keep the JVM flags in `Dockerfile` and `server.tomcat.threads.max=32`. Startup is slow, so avoid adding heavy startup work.
