---
paths:
  - "apps/web/**"
---

# Storefront (`apps/web`)

React 19 + TypeScript, Vite 8, Tailwind 4 (`@tailwindcss/vite`), shadcn-style UI on Radix, TanStack Query, react-hook-form + zod, i18next, Dexie, react-router 7. Run from `apps/web`:
- `npm run dev`
- `npm run build`: runs `tsc -b && vite build`, so type errors fail the build.
- `npm run lint`: oxlint (`.oxlintrc.json`).

## Structure
- Routes: `src/router.tsx`. Every page is wrapped in `AppLayout`, and unknown paths redirect to `/`.
- Pages: `src/pages/<Name>/index.tsx`. Feature components: `components/sections`. Primitives: `components/ui` (shadcn, see `components.json`). Layout: `components/layout`.
- Path alias: `@` → `src`.
- Shared TS types for API DTOs: `src/types/index.ts`. Form schemas: `src/schemas/*` (zod).
- UI strings: `src/locales/en/translation.json` via i18next. Don't hard-code user-facing text.

## API access
- All calls go through `src/api/foodme.ts` → `apiClient` (`src/api/client.ts`), which uses `fetch`. Add new endpoints to `foodmeApi`, not as ad-hoc `fetch` calls.
- Base URL: `VITE_API_BASE_URL` if set. Otherwise `http://localhost:8081` in dev and `""` (relative, same-origin) in production builds.
- The customer JWT is stored by `lib/auth-storage` and sent as a Bearer token. A 401 on `/api/customer/**` clears it. The auth context lives in `providers/auth-provider.tsx`.

## Cart
- The cart is stored client-side in IndexedDB via Dexie: database `FoodMeCart`, table `products` keyed by `uid` and indexed by `chefId` (`src/lib/db.ts`). Logic lives in `src/hooks/useCart.ts`.
- The cart is not sent to the server until checkout. If you change the Dexie schema, add a new `db.version(n)` rather than editing version 1.
- `useCart.ts` is the target of the seeded bug FM-BUG-07 (see the root `.patch`). Check `git diff` before assuming the current behaviour is intended.

## Build and deploy
- `VITE_*` env vars are baked in at build time. In Docker, `VITE_SENTRY_DSN_WEB` is mapped to `VITE_SENTRY_DSN`.
- `lib/sentry.ts` initialises Sentry/GlitchTip (empty DSN = off). `lib/flakyHeartbeat.ts` deliberately reports fake errors; it's demo-only, so leave it.
