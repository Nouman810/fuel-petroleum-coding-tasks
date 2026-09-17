# Task 06: Product Catalog

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Related Ticket:** Task 06 (part of Story Fuel_Petroleum#3)

## Context

Dealers will eventually place orders for fuel and lubricants (Task 07), but there's currently no catalog of what's orderable. This PR adds an Admin-only `Product` catalog — the flat, network-wide list Task 07's ordering flow and Task 10's reporting will read from.

## Approach

**Database & Model:**
- New `Product` model with a `category` enum (`ProductCategory`: `MS` | `HSD` | `LUBRICANT`), mirroring the `EquipmentType` enum precedent from Task 05. Category stays a fixed enum (not free text) because Task 07 (orders) and Task 10 (reporting, product-wise and lubricant-wise breakdowns) both branch on it.
- No relations to any existing table yet — `Order` (Task 07) will add its own `productId` FK when that task lands.
- `Equipment.product` intentionally stays a plain string in this task (confirmed out of scope; deferred to whenever Order's FK pattern is decided).

**Backend Routes (`/products`):**
- Router-level guard: `authenticate, requireRole('ADMIN')` — same shape as `dealers.js` (no `dealerScope`, Product isn't dealer-owned).
- `POST /products`, `GET /products`, `GET /products/:id`, `PATCH /products/:id`, `DELETE /products/:id`.
- `PATCH`/`DELETE` catch Prisma's `P2025` ("record not found") and map to `404`, matching the `dealers.js`/`stations.js` precedent — without it, Express 5's default handler would return an unhandled `500`.
- `price` uses `z.coerce.number().positive()` (bare coercion silently accepts `""` as `0`; always pair coercion with a bound).
- Full CRUD including delete — `Product` currently has zero incoming FKs, so there's no FK-conflict case to guard (unlike `Equipment`'s 409-on-conflict delete).

**Frontend:**
- `src/api/products.js` — CRUD wrappers, same `apiFetch` pattern as `api/dealers.js`/`api/equipment.js`.
- `ProductListPage.jsx` (new) — list + inline create-form (from `DealerListPage.jsx`) combined with per-row inline edit/delete (from `StationEquipmentPage.jsx`'s `EquipmentRow` pattern).
- `AdminShell.jsx` — added `/admin/products` route and a "Manage Products" nav link.
- `vite.config.js` — added `/products` to the dev proxy map.

## Changes

**Backend** (`backend/`)
- `prisma/schema.prisma` — added `ProductCategory` enum and `Product` model.
- `prisma/migrations/20260917060557_add_product/migration.sql` (new) — creates `products` table and `ProductCategory` enum.
- `src/routes/products.js` (new) — all five `/products` route handlers.
- `src/routes/products.test.js` (new, 17 tests) — CRUD, validation, permissions, 404/401/403 coverage.
- `src/app.js` — mounted `productsRouter`.

**Frontend** (`frontend/`)
- `src/api/products.js` (new) — CRUD client.
- `src/routes/ProductListPage.jsx` (new) — product management UI (list, create, inline edit/delete).
- `src/routes/ProductListPage.test.jsx` (new, 4 tests) — rendering and form/action wiring.
- `src/routes/AdminShell.jsx` — added `/admin/products` route and nav link.
- `src/routes/AdminShell.test.jsx` (extended, +2 tests) — route resolution and nav link.
- `vite.config.js` — added `/products` proxy.

## Testing

**Backend:** 17 tests covering create/list/get/update/delete, validation (missing fields, invalid category, empty-string price), permission enforcement (Dealer → 403, unauthenticated → 401), and not-found handling (404 via explicit `P2025` catch, not the framework default `500`).

Run locally: `cd backend && npm test` (requires live Postgres at `DATABASE_URL`)

**Frontend:** 6 tests (4 in `ProductListPage.test.jsx`, 2 in `AdminShell.test.jsx`) covering list rendering, create/edit/delete form wiring, and route/nav resolution.

Run locally: `cd frontend && npm test`

**CI:** `.github/workflows/ci.yml` runs lint + test for both services (backend job includes a live Postgres service container).

## Checklist

- [x] Category is a fixed enum, not free text
- [x] Router-level `requireRole('ADMIN')` guard, no `dealerScope` (Product isn't dealer-owned)
- [x] `PATCH`/`DELETE` map Prisma `P2025` → `404`
- [x] `price` uses `z.coerce.number().positive()` (rejects empty-string coercion to `0`)
- [x] All routes tested (17 backend cases)
- [x] Frontend list/create/edit/delete wired and tested
- [x] Lint passes: `npm run lint` in backend/ and frontend/
- [x] CI verified green on PR
