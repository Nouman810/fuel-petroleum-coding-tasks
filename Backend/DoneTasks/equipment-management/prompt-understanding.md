# Task 05: Equipment Management — Understanding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Depends on:** Task 04 (station management, PR #7, merged)
**Change Class:** FEATURE (new Equipment entity + Dealer CRUD + Admin read-only view; nothing existing changes contract)

## What This Delivers

The equipment hierarchy (tanks, dispensers, nozzles) that Task 09's shift-closing sales will record against per nozzle. Dealer-facing full CRUD, scoped to their own stations; Admin gets read-only visibility across the whole network.

**Backend (`backend/`):**
- A single `Equipment` Prisma model with a `type` discriminator (`TANK`|`DISPENSER`|`NOZZLE`), matching spec.md's ERD exactly (`spec.md` lines 79-87) — the spec models one polymorphic `EQUIPMENT` entity, not three separate tables, so that's what gets built. Fields: `id`, `stationId` (required — every equipment row, regardless of type, is directly tied to a station per the ERD), `type`, `parentEquipmentId` (nullable self-relation — a nozzle's attaching dispenser; null for tanks/dispensers), `identifier`, `capacity` (nullable, tanks only), `currentMeterReading` (nullable, nozzles only).
- `product` field added to `Equipment` (confirmed with user) — a plain string, not a FK. The ERD has no product field at all on Equipment, and Product Catalog (Task 06) doesn't exist yet to reference. This mirrors the exact precedent already set in this project: `User.dealerId` was a plain nullable column in Task 02 until `Dealer` existed in Task 03, then got a real FK. `product` follows the same path — plain text now, a later task converts it to a real FK once `Product` exists.
- `POST /equipment` (Dealer only) — creates one equipment row. Request shape depends on `type`:
  - `TANK`: `stationId`, `identifier`, `capacity`, `product` (all required)
  - `DISPENSER`: `stationId`, `identifier` (required)
  - `NOZZLE`: `stationId`, `identifier`, `parentEquipmentId`, `currentMeterReading` (all required) — `currentMeterReading` is the dealer-supplied baseline at creation time, per the raw prompt ("starts at whatever the dealer records as the baseline when the nozzle is added").
  - The `stationId` in the request must belong to the calling dealer (checked directly — `dealerScope` resolves ownership of an *existing* `:id` resource, which doesn't apply to a creation request with no id yet). For a `NOZZLE`, `parentEquipmentId` must reference an existing `DISPENSER` at that same station.
- `GET /equipment` — Admin: all equipment, optional `?stationId=` filter. Dealer: scoped via a Prisma relation filter (`where: { station: { dealerId: req.user.dealerId } }`), which safely combines with an optional `?stationId=` filter the dealer sends (their own station, to narrow a multi-station list) — the relation filter still holds even if a dealer tried passing another dealer's station id, since Prisma ANDs the two conditions.
- `GET /equipment/:id` — reuses the `dealerScope` middleware from Task 02/04 (`backend/src/middleware/auth.js`), resolving ownership by walking `equipment → station → dealerId` (equipment doesn't carry `dealerId` directly). Admin bypasses via `dealerScope`'s existing non-DEALER bypass, matching "Admin can view... across all stations."
- `PATCH /equipment/:id` and `DELETE /equipment/:id` — gated with `requireRole('DEALER')` **in addition to** `dealerScope`, because `dealerScope` alone treats any non-DEALER caller as trusted and lets them through (that's exactly right for the read-only `GET`, but wrong here — the raw prompt is explicit that Admin must NOT be able to edit). `PATCH`'s editable fields depend on the existing row's `type` (fetched first): `identifier` for all types, plus `capacity`/`product` for `TANK` only. `type`, `stationId`, `parentEquipmentId`, and `currentMeterReading` are never PATCHable — reassigning equipment to a different station/parent is out of scope (mirrors Station's "not reassignment" rule from Task 04), and `currentMeterReading` is framed by the raw prompt as a value Task 09's shift-sales recording will update going forward, not something freely editable here.
- `DELETE /equipment/:id` on a `DISPENSER` that still has `NOZZLE` children hitting the self-relation FK constraint → caught and returned as `409` (matches spec.md's Error Handling section, "business-rule conflict"), not a raw 500.
- Validation via `zod` (`discriminatedUnion` on `type` for `POST`), matching the established pattern. `404` for a nonexistent `stationId` on create, or a nonexistent `parentEquipmentId` / one that isn't a `DISPENSER` at the same station.

**Frontend (`frontend/`):**
- `DealerShell` gains an "Equipment" section under a Dealer's station — nested under the existing `MyStationsPage` context (a Dealer manages equipment *per station*, and per the raw prompt/spec a dealer can have zero, one, or multiple stations, an invariant already handled in Task 04). CRUD forms for tanks/dispensers/nozzles, station-scoped.
- `AdminShell` gains a read-only equipment view (list only, no create/edit forms) filterable by station, for support/troubleshooting per the raw prompt.
- Both go through the existing `apiFetch` client; `vite.config.js`'s dev proxy needs an `/equipment` entry (same gap class Tasks 03/04 already hit for `/dealers` and `/stations`).

## Out of Scope

- A real `Product` foreign key on `Equipment.product` — deferred to Task 06 or later, per the confirmed decision above.
- Anything Task 09 (shift-sales) does with `currentMeterReading` — this task only establishes the field and its creation-time baseline.
- Admin editing equipment — explicitly read-only per the raw prompt.

## Key Decisions (confirmed with user)

- **`Equipment.product`:** plain string field, no FK (Product Catalog doesn't exist yet). Mirrors the `User.dealerId` → `Dealer` FK precedent from Tasks 02/03.

## Applicable Rules

`standards_location` is unset in `.claude/config_hints.json` (no project-specific coding standards installed). Follow the same Express/Prisma/React/zod conventions established in Tasks 01-04.

## Branching Strategy

This task is part of a multi-PR story. Branch from and target PRs to the story branch:
- **Story branch:** story/3-dealer-fuel-station-management-system
- **Feature branch:** feature/fuel_petroleum-XXX-equipment-management (branched from story/3-dealer-fuel-station-management-system, ticket-late — same convention as Tasks 01-04, ticket number filled in at Phase 4i)
- **PR target:** story/3-dealer-fuel-station-management-system (not main)

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
