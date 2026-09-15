# Task 05: Equipment Management

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)  
**Related Ticket:** Task 05 (part of Story Fuel_Petroleum#3)

## Context

Dealers and admins need to manage physical fuel station equipment (tanks, dispensers, nozzles) as the infrastructure foundation for transactions. This PR adds a polymorphic Equipment entity with dealer-scoped CRUD and admin read-only visibility, reusing the established `dealerScope` middleware pattern from earlier tasks.

## Approach

**Database & Model:**
- Single `Equipment` Prisma model with a `type` discriminator (`TANK`|`DISPENSER`|`NOZZLE`), matching spec.md's ERD exactly, not three separate tables.
- Self-relation (`parentEquipmentId`) for nozzle-to-dispenser hierarchy, with explicit `onDelete: Restrict` (not Prisma's default `SET NULL`) to prevent silent orphaning.
- `stationId` directly on every Equipment row for single-filter scoping (no recursive hierarchy walk).
- `product` as a plain string field (no FK) — mirrors the `User.dealerId` precedent from Tasks 02/03; becomes a real FK once Product Catalog (Task 06) exists.
- Decimal fields (`capacity`, `currentMeterReading`) for precision; test assertions compare `Number(...)` since Prisma serializes decimals as JSON strings.

**Backend Routes (`/equipment`):**
- `POST /equipment` (Dealer only) — creates equipment with type-specific validation via `z.discriminatedUnion`. For `NOZZLE`, validates `parentEquipmentId` references a `DISPENSER` at the same station. Uses `z.coerce.number()` (not bare `z.number()`) since form submissions produce strings; `.positive()` for capacity, `.nonnegative()` for nozzles (zero baseline is valid).
- `GET /equipment` (Admin: all; Dealer: scoped via relation filter `where: { station: { dealerId } }`). Optional `?stationId=` filter AND'd with relation filter — a dealer passing another dealer's station still yields empty array (not their own equipment anyway).
- `GET /equipment/:id` — reuses `dealerScope` middleware, resolving ownership via `equipment → station → dealerId`.
- `PATCH /equipment/:id` (Dealer only) — type-specific schemas: `identifier` + `capacity`/`product` for tanks, `identifier`-only for dispensers/nozzles. Immutable fields (`type`, `stationId`, `parentEquipmentId`, `currentMeterReading`) stripped by zod's default unknown-key handling.
- `DELETE /equipment/:id` (Dealer only) — catches Prisma `P2003` (FK constraint on nozzle children) → `409` Conflict. Returns `200` with deleted row JSON (consistent with every other `api/*.js` wrapper).

**Access Control:**
- `POST /equipment` checks `stationId` directly belongs to the calling dealer (not reachable via `dealerScope`, which resolves `:id` resources only).
- `PATCH`/`DELETE` use `requireRole('DEALER')` *before* `dealerScope` — because `dealerScope` alone treats non-DEALER (Admin) as trusted and lets them through (correct for read-only `GET`, wrong for mutation).

**Frontend:**
- `src/api/equipment.js` — CRUD wrappers (`listEquipment`, `createEquipment`, `updateEquipment`, `deleteEquipment`), same `apiFetch` pattern as dealers/stations.
- `StationEquipmentPage.jsx` (new) — per-station equipment list with type-grouped display, create forms (type-specific fields), and inline edit/delete.
- `AdminEquipmentPage.jsx` (new) — read-only list with station filter picker (no create/edit/delete rendered).
- `MyStationsPage.jsx` — added "Equipment" link on each station card (`{station.id}/equipment`), the sole click-path to equipment for dealers.
- `AdminHome` — added "Manage Equipment" nav link (`/admin/equipment`).
- Route wiring in `DealerShell.jsx` and `AdminShell.jsx`.
- **Important:** `MyStationsPage.test.jsx` tests updated to render inside `<MemoryRouter>` — required now that the component uses `<Link>`.

## Changes

**Backend** (`backend/`)
- `prisma/schema.prisma` — added `EquipmentType` enum and `Equipment` model with self-relation, `onDelete: Restrict`; added `equipment` relation to `Station`.
- `prisma/migrations/20260915140000_add_equipment/migration.sql` (new) — creates `equipment` table and `EquipmentType` enum; two FK constraints both `RESTRICT`.
- `src/routes/equipment.js` (new, 161 LOC) — all five `/equipment` route handlers with type-specific validation and scoping.
- `src/routes/equipment.test.js` (new, 372 LOC) — 30+ test cases covering CRUD, permissions, validation, scoping, and FK conflicts.
- `src/app.js` — mounted `equipmentRouter`.

**Frontend** (`frontend/`)
- `src/api/equipment.js` (new) — CRUD client.
- `src/routes/StationEquipmentPage.jsx` (new, 267 LOC) — equipment management UI for dealers.
- `src/routes/StationEquipmentPage.test.jsx` (new, 111 LOC) — component and form tests.
- `src/routes/AdminEquipmentPage.jsx` (new, 103 LOC) — read-only equipment view.
- `src/routes/AdminEquipmentPage.test.jsx` (new, 69 LOC) — verification of read-only rendering and absence of edit controls.
- `src/routes/MyStationsPage.jsx` — added Equipment link per station.
- `src/routes/MyStationsPage.test.jsx` — wrapped existing tests with `<MemoryRouter>`, added Equipment link assertion.
- `src/routes/DealerShell.jsx` — added `stations/:stationId/equipment` route.
- `src/routes/DealerShell.test.jsx` (extended) — new test for route resolution, added `api/equipment` mock.
- `src/routes/AdminShell.jsx` — added `/equipment` route and "Manage Equipment" link.
- `src/routes/AdminShell.test.jsx` (extended) — two new tests: `/admin/equipment` resolves, and AdminHome renders nav link.
- `frontend/vite.config.js` — added `/equipment` proxy to backend.

## Testing

**Backend:** 30+ unit tests covering:
- Type-specific creation (tanks, dispensers, nozzles)
- Validation (malformed bodies, invalid parents, missing stations)
- Permission enforcement (dealer/admin, cross-dealer rejection)
- Scoping (relation filter + optional same-dealer station filter)
- Mutations (PATCH field-scoping per type, DELETE with FK constraint caught as 409)
- Authentication (unauthenticated 401)

Run locally: `cd backend && npm test` (requires live Postgres at `DATABASE_URL`)

**Frontend:** Component and integration tests covering:
- Equipment list display grouped by type
- Type-specific create forms
- Edit/delete UX and API calls
- Admin read-only (no controls rendered)
- Route resolution and nav links
- Mocked API to avoid hitting jsdom fetch

Run locally: `cd frontend && npm test`

**CI:** `.github/workflows/ci.yml` runs lint + test for both services on PR (backend job includes Postgres service container).

## Checklist

- [x] Follows polymorphic pattern and spec.md's ERD exactly
- [x] Reuses `dealerScope` middleware; no logic re-derived
- [x] Type-specific schemas via zod `discriminatedUnion`
- [x] Dealer mutations require `requireRole('DEALER')` + `dealerScope`
- [x] Decimal fields tested with `Number(...)` comparison
- [x] Form coercion via `z.coerce.number()` with bounds
- [x] Frontend MemoryRouter wrapping fixed for `<Link>` usage
- [x] All routes tested; no dead code (30+ test scenarios)
- [x] Lint passes: `npm run lint` in backend/ and frontend/
- [x] CI verified green on PR
