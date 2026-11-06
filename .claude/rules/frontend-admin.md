---
paths:
  - "apps/admin/**"
---

# Admin back office (`apps/admin`)

react-admin 5 + MUI 6, React 18, react-router 6, axios. Plain JS/JSX, no TypeScript. Run from `apps/admin`:
- `npm run dev`
- `npm run build`
- `npm run lint`: eslint.

## Structure
- `src/App.jsx`: react-admin `<Admin>` with the resources `orders`, `chefs` and `dishes`. Their pages are in `src/pages/<resource>/`.
- `src/providers/dataProvider.js`: adapts the backend to react-admin.
  - Resource names are plural, but backend paths are singular (`RESOURCE_TO_PATH`: `orders` → `/order`, …). Register any new resource there.
  - The backend returns `{ list, count }`, which is mapped to `{ data, total }`. Pages are 0-based on the backend and 1-based in react-admin.
  - Sorting is done **client-side**, over the current page only.
- `src/providers/authProvider.js` and `src/api/auth-api.js`: login through `POST /admin/auth/login` (JWT). `security/ProtectedRoute` and `GuestRoute` guard the routes.
- `src/api/base-api.js`: the axios instance. Per-resource API modules sit beside it.
- `src/constants/OrderStatus.jsx`: order status values. Keep them in sync with the backend.

## Base path and deploy
- In production the app is served by the backend under **`/backoffice/`**. `vite.config.js` sets `base` only in production mode, while dev runs at `/`. `/admin/**` is the backend API, so don't route the UI there.
- `VITE_API_BASE_URL` and `VITE_SENTRY_DSN` are baked in at build time. In Docker, `VITE_SENTRY_DSN_ADMIN` is mapped to `VITE_SENTRY_DSN`.
- `lib/flakyHeartbeat.js` deliberately reports fake errors; it's demo-only.
