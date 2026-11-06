---
paths:
  - "apps/web/e2e/**"
  - "apps/admin/e2e/**"
  - "apps/web/playwright.config.ts"
  - "apps/admin/playwright.config.ts"
---

# Frontend testing (Playwright e2e — web & admin)

Both apps use Playwright with Chromium only. There are no unit tests; e2e is the frontend test suite. **The backend must be running on :8081** (with Postgres) because the tests hit the real API, and that includes the simulated 200–1500 ms latency. Use web-first assertions and auto-waiting, never fixed sleeps.

## Storefront (`apps/web`)
- `npm run test:e2e`: Playwright starts `npm run dev -- --port 5180` itself, or reuses an already-running server outside CI. Override the target with `PLAYWRIGHT_BASE_URL`.
- To run one spec: `npx playwright test e2e/happy-path.spec.ts`. Add `-g "<title>"` for one test, and `--headed` or `--ui` to debug.
- `npm run test:e2e:all`: runs the web suite, then the admin suite.
- Tests run in parallel (`fullyParallel`), so each test must create its own data.
- `e2e/auth.ts` has the shared helpers:
  - `createAccountAtCheckout` registers through the UI.
  - `registerCustomerViaApi` registers through `request`.
  - Both generate unique emails with `Date.now()` and a random number.
- Selectors: prefer roles and labels (`getByRole`, `getByLabel`), matching the accessible names the app renders from `translation.json`.
- `flake-*.spec.ts` (`flake-cart-persistence`, `flake-dish-modal`) are **intentionally flaky** course exercises. Don't delete them or quietly "stabilise" them unless asked.
- The cart lives in IndexedDB, so cart-persistence tests depend on browser storage within a context.

## Admin (`apps/admin`)
- `npm run test:e2e`: Playwright starts the dev server on :5174 (`--strictPort`). Override with `ADMIN_BASE_URL`.
- Tests run **serially** (`fullyParallel: false`) because they share admin state and data. Specs live in `e2e/admin-flows.spec.ts`.
- In dev the app runs at `/`, not `/backoffice/`.

## Both
- In CI: `forbidOnly` is on, with retries (web 2, admin 1) and traces captured on the first retry. Reports go to `playwright-report/` as HTML.
