# Execution Plan: Task 05 — Equipment Management

**Branch:** `feature/fuel_petroleum-XXX-equipment-management` (branched from `story/3-dealer-fuel-station-management-system`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system`
**Change Class:** FEATURE

## Summary

Add the `Equipment` entity — a single polymorphic table (`TANK`/`DISPENSER`/`NOZZLE`) matching the spec's ERD exactly. Dealer gets full CRUD scoped to their own stations, reusing the `dealerScope` middleware; Admin gets read-only visibility across the whole network.

## Approach & Trade-offs

- **One `Equipment` model with a `type` discriminator, not three separate tables.** The spec's ERD (`spec.md` lines 79-87) explicitly draws one `EQUIPMENT` entity with `type: TANK|DISPENSER|NOZZLE`, `parentEquipmentId` (nozzle-to-dispenser), `identifier`, `capacity` ("tanks only"), `currentMeterReading` ("nozzles only"). Building three separate tables would deviate from the documented data model without cause.
- **`product` is a plain nullable string, not a FK**, confirmed with the user — the ERD has no product field at all, and `Product` (Task 06) doesn't exist yet. Mirrors the `User.dealerId` precedent (Task 02: plain column → Task 03: real FK once `Dealer` existed).
- **`parentEquipmentId`'s self-relation FK is explicitly `onDelete: Restrict`, not Prisma's default.** Prisma defaults an *optional* relation's `onDelete` to `SET NULL` — confirmed by generating the migration once without an explicit `onDelete` and observing exactly that. For this relation, `SET NULL` would silently orphan a nozzle (leave it pointing at no dispenser) the moment its dispenser is deleted, with no error and no trace. Explicitly setting `onDelete: Restrict` makes deleting a dispenser with existing nozzles fail at the DB level instead — caught in the `DELETE` handler and surfaced as `409`, consistent with spec.md's Error Handling section ("business-rule conflict").
- **`stationId` is required on every `Equipment` row, regardless of `type`.** The ERD puts `stationId` directly on `EQUIPMENT`, not only reachable by walking through a dispenser — this makes `GET /equipment` scoping (below) a single relation filter instead of a recursive walk up the hierarchy.
- **Dealer-scoping reuses `dealerScope` from Task 02, applied at `equipment → station → dealerId`** (equipment carries no `dealerId` directly, so the ownership resolver joins through `station`), per the raw prompt's explicit instruction not to re-derive the scoping logic.
- **`POST /equipment` cannot use `dealerScope`** (it resolves ownership of an *existing* `:id` resource; creation has no id yet). Instead the handler checks the request's `stationId` directly belongs to the calling dealer, mirroring how `POST /dealers` and `POST /stations` each handle their own creation-time checks rather than forcing everything through `dealerScope`.
- **`GET /equipment`'s Dealer scoping combines a relation filter with an optional same-dealer `stationId` narrow:** `where: { station: { dealerId: req.user.dealerId } }`, ANDed with `stationId` from the query if present. Since Prisma ANDs a scalar `stationId` filter with the nested relation filter in the same `where`, a Dealer passing another dealer's `stationId` still yields zero rows — the relation filter can't be bypassed by the scalar one.
- **`PATCH`/`DELETE /equipment/:id` are gated with `requireRole('DEALER')` in *addition* to `dealerScope`.** `dealerScope` alone treats any non-`DEALER` caller as trusted and lets them straight through — correct for the read-only `GET` (that's exactly how Admin gets its cross-network view), wrong here, since the raw prompt is explicit that Admin must not be able to edit or delete. `requireRole('DEALER')` closes that gap before `dealerScope` even runs.
- **`PATCH`'s editable-field set depends on the existing row's `type`, fetched first.** `identifier` is editable for every type; `capacity`/`product` only for `TANK`. `type`, `stationId`, `parentEquipmentId`, and `currentMeterReading` are never PATCHable — reassigning equipment to a different station/parent is out of scope (mirrors Station's Task-04 "not reassignment" rule), and `currentMeterReading` is framed by the raw prompt as a value Task 09's shift-sales flow will update going forward, not something freely editable here. zod's default unknown-key stripping (already relied on in `dealers.js`/`stations.js`) enforces this: a `PATCH` body including `currentMeterReading` or `type` simply has those fields dropped before they ever reach `prisma.equipment.update`.
- **`POST /equipment` validates the type-specific shape via `zod`'s `discriminatedUnion('type', [...])`** — three literal-`type` sub-schemas (`TANK`, `DISPENSER`, `NOZZLE`), each requiring exactly the fields that type needs.
- **A `NOZZLE`'s `parentEquipmentId` must reference an existing `DISPENSER` at the *same* station** — checked explicitly in the handler (not just "does this id exist as any equipment row"), `404` if it doesn't resolve to a same-station dispenser.
- **`capacity` and `currentMeterReading` use `z.coerce.number().positive()` / `.nonnegative()`, not bare `z.number()`.** Every existing create/edit form in this codebase (`StationListPage.jsx`'s `handleChange`, `DealerListPage.jsx`'s equivalent) produces plain strings from `event.target.value` — there is no established precedent here for sending a real JS number from a form. A strict `z.number()` would `400` on every real form submission even though a hand-built unit-test payload (already numeric) passes. Coercing on the backend keeps the frontend form code identical to every other form in the app. The bare `.positive()`/`.nonnegative()` bound matters on its own: `z.coerce.number()` alone silently turns an empty-string form field into `0` (confirmed: `''` coerces to `0` without error) rather than rejecting a required-but-unfilled field — `capacity` (tanks) uses `.positive()` since a zero-capacity tank makes no sense, `currentMeterReading` (nozzles) uses `.nonnegative()` since a fresh nozzle's baseline can legitimately be `0`.
- **Decimal fields serialize to JSON as strings, not numbers** — confirmed by exercising the installed `@prisma/client` runtime directly (`JSON.stringify` on a `Decimal` yields `"1000"`, not `1000`, with trailing zeros dropped). Every test asserting `capacity`/`currentMeterReading` compares `Number(res.body.field)`, not a raw `toBe(1000)`, and the frontend reads/writes them as strings through the same form-value pattern every other field already uses.
- **`DELETE /equipment/:id` returns `200` with the deleted row's JSON**, not a bare `204`. Every existing `api/*.js` wrapper (`stations.js`, `dealers.js`) ends in `return res.json()` — a `204` has no body and `res.json()` would throw. Returning the deleted record keeps `deleteEquipment` consistent with every other wrapper instead of being the one function with different response-handling code.
- **Teardown for `equipment.test.js` uses one `prisma.equipment.deleteMany({ where: { id: { in: [...] } } })` call, not per-row deletes.** With `ON DELETE RESTRICT` on the self-relation, ordering *would* matter for per-row deletes (a loop deleting a dispenser before its nozzle would fail) — confirmed against the live DB that a single multi-id `deleteMany` succeeds because Postgres evaluates the RI constraint as an end-of-statement check, not per-row. This is the opposite of "ordering doesn't matter because of Restrict" — it's "ordering doesn't matter *because it's one statement*, despite Restrict." Any test or cleanup code that deletes equipment rows individually (e.g. the in-test DELETE-conflict scenario, see Test Plan) must still go children-first.

## Files to Change

**Backend** (`backend/`)
- `prisma/schema.prisma` — add `EquipmentType` enum and `Equipment` model (self-relation with explicit `onDelete: Restrict`); add `equipment Equipment[]` to `Station`. See Database Schema below.
- `prisma/migrations/20260915140000_add_equipment/migration.sql` — new, generated via `npx prisma migrate diff --from-schema-datamodel <current> --to-schema-datamodel <draft> --script` (real tool output, see Database Schema). Timestamped after Task 04's `20260915130000_add_station`.
- `src/routes/equipment.js` — new. `router.use('/equipment', authenticate)` (every route needs a session; role-specific access enforced per-route, same pattern as `stations.js`):
  - `POST /equipment` — `requireRole('DEALER')`. zod `discriminatedUnion` on `type`. Verifies `stationId` belongs to the caller (`prisma.station.findUnique`, `404` if missing, `403` if it belongs to a different dealer). For `NOZZLE`, verifies `parentEquipmentId` resolves to a `DISPENSER` at the same `stationId` (`404` otherwise). `201` on success.
  - `GET /equipment` — Admin: all, optional `?stationId=` filter. Dealer: `403` if `req.user.dealerId` is falsy (same guard class as `stations.js`'s list endpoint); otherwise scoped via the relation filter described above.
  - `GET /equipment/:id` — `dealerScope(getEquipmentOwnerDealerId)`, where the resolver is `(await prisma.equipment.findUnique({ where: { id }, include: { station: true } }))?.station?.dealerId`. `404` if not found (Admin path; Dealer path 403s first via `dealerScope` for a mismatched owner).
  - `PATCH /equipment/:id` — `requireRole('DEALER')`, then `dealerScope(...)`. Fetches the existing row to determine `type`, builds the type-appropriate zod schema, updates only those fields.
  - `DELETE /equipment/:id` — `requireRole('DEALER')`, then `dealerScope(...)`. Catches Prisma `P2003` (FK violation — a `DISPENSER` with existing `NOZZLE` children) → `409`. (No `P2025` catch: `dealerScope` already rejects a nonexistent id with `403` before the handler runs, for the only caller — `DEALER` — that can reach this route at all, so that branch would be dead code.) `200` with the deleted row's JSON on success (see Approach).
- `src/routes/equipment.test.js` — new, see Test Plan.
- `src/app.js` — mount `equipmentRouter` (`require('./routes/equipment')`, `app.use(equipmentRouter)`), same pattern as `stationsRouter`.

**Frontend** (`frontend/`)
- `vite.config.js` — add `'/equipment': 'http://localhost:3000'` to `server.proxy` (same gap class Tasks 03/04 each hit for `/dealers`/`/stations`).
- `src/api/equipment.js` — new: `listEquipment(params)` (`{ stationId }`), `getEquipment(id)`, `createEquipment(payload)`, `updateEquipment(id, payload)`, `deleteEquipment(id)`. Same `apiFetch`-wrapping pattern as `api/dealers.js`/`api/stations.js`.
- `src/routes/MyStationsPage.jsx` — each station card gets an "Equipment" `<Link to={`${station.id}/equipment`}>` routing to that station's equipment page (a Dealer manages equipment *per station*, and Task 04 already established that a Dealer can have zero/one/many stations). This is the sole click-path to equipment on the Dealer side (see `DealerShell.jsx` below).
- `src/routes/MyStationsPage.test.jsx` — **modify (not just add elsewhere):** all three existing `render(<MyStationsPage />)` calls must move to `render(<MemoryRouter><MyStationsPage /></MemoryRouter>)`. Confirmed empirically: `MyStationsPage.jsx` currently imports nothing from `react-router-dom`, which is exactly why the bare `render()` calls work today — the moment it gains a `<Link>`, a bare render throws (`Cannot destructure property 'basename' of 'React.useContext(...)' as it is null`) with no Router ancestor. This is a Files-to-Change item in its own right, not a side-effect left implicit.
- `src/routes/DealerShell.jsx` — add a `stations/:stationId/equipment` route only. No change to `DealerHome` — the click-path to equipment is entirely through `MyStationsPage`'s per-station "Equipment" link (added above), since equipment is inherently per-station and `DealerHome` has no station context to link into.
- `src/routes/StationEquipmentPage.jsx` — new: fetches equipment for one station (`listEquipment({ stationId })`), renders tanks/dispensers/nozzles with create forms (type-specific fields) and inline edit/delete.
- `src/routes/StationEquipmentPage.test.jsx` — new, see Test Plan.
- `src/routes/AdminShell.jsx` — add an `equipment` route (read-only list, `?stationId=` filter via a dealer→station picker, no create/edit/delete controls rendered at all), **and** add a "Manage Equipment" `<Link>` in `AdminHome` alongside the existing "Manage Dealers"/"Manage Stations" links (`AdminHome` already renders one `<Link>` per section — adding the route without the link leaves `/admin/equipment` reachable only by typing the URL). Verified by a new `AdminShell.test.jsx` case (see Test Plan) that renders `/admin` and asserts the "Manage Equipment" link is present — not just that `/admin/equipment` itself resolves, which is a separate assertion.
- `src/routes/AdminEquipmentPage.jsx` — new: read-only list, station filter.
- `src/routes/AdminEquipmentPage.test.jsx` — new, see Test Plan.
- `src/routes/AdminShell.test.jsx` — extend: add `vi.mock('../api/equipment', ...)` alongside the existing `api/dealers`/`api/stations` mocks (the exact gap `a_sag_code_reviewer` caught in Task 04 when `AdminShell.test.jsx` needed a new mock for a newly-rendered child route). New case asserting `/admin/equipment` resolves to `AdminEquipmentPage`.
- `src/routes/DealerShell.test.jsx` — extend: **also** add `vi.mock('../api/equipment', ...)` (currently mocks only `api/stations` — rendering `StationEquipmentPage` unmocked would hit the real `apiFetch` against jsdom, the identical gap class as the AdminShell.test.jsx fix above, just easy to miss on the Dealer side). New case asserting `/dealer/stations/:stationId/equipment` resolves to `StationEquipmentPage`.

**Root**
- No `.github/workflows/ci.yml` changes needed — no new env vars, no new services.
- No `README.md` changes needed — no new local-setup steps.

## Test Plan

**Backend** (`equipment.test.js`) — creates its own fixture Admin, a fixture Dealer A with a station and a `DEALER` user/token, a fixture Dealer B with a station and a `DEALER` user/token, and a fixture orphan `DEALER` user with no linked `dealerId` (mirroring `stations.test.js`'s `FIXTURE_ORPHAN_DEALER_EMAIL`/`orphanDealerToken` pattern), against the live Postgres. **Teardown:** one `prisma.equipment.deleteMany({ where: { id: { in: [...] } } })` call for all created equipment ids (safe regardless of parent/child order — see Approach) → `user.deleteMany` → `station.deleteMany` → `dealer.deleteMany`. **All `capacity`/`currentMeterReading` assertions compare `Number(res.body.field)`, never a raw `toBe(1000)`** — Decimal fields serialize as strings (see Approach).

- `POST /equipment` (Dealer A, `type: TANK`) with a valid `stationId` (their own) → `201`, `Number(capacity)` matches, `product` present, `parentEquipmentId` null
- `POST /equipment` (Dealer A, `type: DISPENSER`) → `201`, `capacity`/`product`/`currentMeterReading` all null
- `POST /equipment` (Dealer A, `type: NOZZLE`) with `parentEquipmentId` set to the dispenser just created → `201`, `Number(currentMeterReading)` equals the supplied baseline
- `POST /equipment` with a malformed body (missing `identifier`) → `400`
- `POST /equipment` (Dealer A) with a `stationId` belonging to Dealer B → `403`
- `POST /equipment` (Dealer A, `type: NOZZLE`) with a `parentEquipmentId` that isn't a `DISPENSER` at the same station → `404`
- `POST /equipment` (Admin token) → `403` (the `requireRole('DEALER')` guard blocks Admin from creating, not just from editing)
- `GET /equipment` (Admin) → includes all fixture equipment across both dealers
- `GET /equipment` (Admin) with `?stationId=` → only that station's equipment
- `GET /equipment` (Dealer A) → only Dealer A's station's equipment
- `GET /equipment` (Dealer A) with `?stationId=<Dealer A's own station>` → their equipment (the scalar filter narrows within what the relation filter already allows)
- `GET /equipment` (Dealer A) with `?stationId=<Dealer B's station>` → `200` with an **empty array** — the relation filter (`station.dealerId = Dealer A`) is ANDed with the scalar filter (`stationId = Dealer B's station`), and no row satisfies both, so the correct assertion is "nothing comes back," not "Dealer A's own equipment comes back anyway" (the latter would be true under Task 04's `stations.js` list pattern, which ignores the query param entirely for a Dealer — this endpoint does not; the two are genuinely different designs, don't cross-copy the assertion)
- `GET /equipment` (orphan Dealer token, no `dealerId`) → `403`
- `GET /equipment/:id` (Admin) for a non-existent id → `404`
- `GET /equipment/:id` (Dealer B) on Dealer A's tank → `403`
- `PATCH /equipment/:id` (Dealer A) on their tank: body includes `capacity`, `product`, AND `currentMeterReading`/`type`/`stationId` → `200`, only `capacity`/`product` changed, the rest untouched (proves the type-scoped schema strips them, not just that valid fields work)
- `PATCH /equipment/:id` (Dealer A) on their dispenser with a `capacity` in the body → `200`, `capacity` remains `null` (proves the type-scoped schema, not the tank one, applied)
- `PATCH /equipment/:id` (Dealer A) on their nozzle: body includes `identifier` AND `currentMeterReading` → `200`, `identifier` changed, `currentMeterReading` unchanged (the third type-scoped schema — untested would mean a bug here ships silently)
- `PATCH /equipment/:id` (Dealer B) on Dealer A's equipment → `403`
- `DELETE /equipment/:id` (Dealer B) on Dealer A's equipment → `403`
- `PATCH /equipment/:id` and `DELETE /equipment/:id` (Admin token) on any equipment → `403` for both (one test, two assertions — proves the shared `requireRole('DEALER')` guard on both routes, same "one row, two route shapes" pattern as Task 04's AC-9)
- `DELETE /equipment/:id` (Dealer A) on their dispenser while its nozzle still exists → `409`
- `DELETE /equipment/:id` (Dealer A) on the nozzle, then the now-childless dispenser (two separate calls, children-first) → both `200`, each body is the deleted row
- `GET /equipment` with no `Authorization` header → `401`; `GET /equipment/:id` with no header → `401` too (both route shapes, same lesson as Task 04's AC-9)

**Frontend:**
- `MyStationsPage.test.jsx` — the three existing tests updated to render inside `<MemoryRouter>` (required once the component gains a `<Link>`, see Files to Change); no new test cases, just the wrapper fix
- `StationEquipmentPage.test.jsx` — mocks `api/equipment.js`: renders tanks/dispensers/nozzles grouped or listed with type visible; submitting each type's create form calls `createEquipment` with the type-correct payload shape; deleting calls `deleteEquipment`
- `AdminEquipmentPage.test.jsx` — mocks `api/equipment.js` **and `api/stations.js`** (the station picker fetches stations the same way `StationListPage.jsx`'s dealer picker fetches dealers — an unmocked `api/stations` call would hit the real `apiFetch` in jsdom): renders equipment across stations, no create/edit/delete controls present in the rendered output (asserts their absence, not just that reads work)
- `AdminShell.test.jsx` — two new cases: `/admin/equipment` resolves to `AdminEquipmentPage`; `/admin` (the home page) renders a "Manage Equipment" link (proves the nav entry itself, not just that the route resolves once reached)
- `DealerShell.test.jsx` — new case: `/dealer/stations/:stationId/equipment` resolves to `StationEquipmentPage`
- `MyStationsPage.test.jsx` — one more assertion (alongside the MemoryRouter fix above): a rendered station card includes an "Equipment" link pointing at `{station.id}/equipment` — this is the only click-path to equipment on the Dealer side, so it needs its own proof, not just the destination route resolving

**CI:** both suites run via the existing `.github/workflows/ci.yml` jobs, no changes needed there. This dev machine's local Postgres (set up during Task 04) is still running — the backend suite will be verified locally in Phase 3/4 too, not only via CI.

## Database Schema

Added to `prisma/schema.prisma`:

```prisma
enum EquipmentType {
  TANK
  DISPENSER
  NOZZLE
}

model Equipment {
  id                  String        @id @default(uuid())
  stationId           String
  station             Station       @relation(fields: [stationId], references: [id])
  type                EquipmentType
  parentEquipmentId   String?
  parentEquipment     Equipment?    @relation("EquipmentParent", fields: [parentEquipmentId], references: [id], onDelete: Restrict)
  children            Equipment[]   @relation("EquipmentParent")
  identifier          String
  capacity            Decimal?      @db.Decimal(12, 2)
  product             String?
  currentMeterReading Decimal?      @db.Decimal(12, 2)
  createdAt           DateTime      @default(now())
  updatedAt           DateTime      @updatedAt

  @@map("equipment")
}
```

`Station` gains the inverse relation:
```prisma
model Station {
  id          String      @id @default(uuid())
  dealerId    String
  dealer      Dealer      @relation(fields: [dealerId], references: [id])
  name        String
  address     String
  region      String
  contactInfo String
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  equipment   Equipment[]

  @@map("stations")
}
```

Migration SQL (generated via `npx prisma migrate diff --from-schema-datamodel ./prisma/schema.prisma --to-schema-datamodel <draft-schema-with-the-above> --script`, run against the current committed schema — no live DB connection needed for this command; real tool output, regenerated once without the explicit `onDelete: Restrict` to confirm Prisma's optional-relation default really is `SET NULL`, then with it to get the SQL below):

```sql
-- CreateEnum
CREATE TYPE "EquipmentType" AS ENUM ('TANK', 'DISPENSER', 'NOZZLE');

-- CreateTable
CREATE TABLE "equipment" (
    "id" TEXT NOT NULL,
    "stationId" TEXT NOT NULL,
    "type" "EquipmentType" NOT NULL,
    "parentEquipmentId" TEXT,
    "identifier" TEXT NOT NULL,
    "capacity" DECIMAL(12,2),
    "product" TEXT,
    "currentMeterReading" DECIMAL(12,2),
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "equipment_pkey" PRIMARY KEY ("id")
);

-- AddForeignKey
ALTER TABLE "equipment" ADD CONSTRAINT "equipment_stationId_fkey" FOREIGN KEY ("stationId") REFERENCES "stations"("id") ON DELETE RESTRICT ON UPDATE CASCADE;

-- AddForeignKey
ALTER TABLE "equipment" ADD CONSTRAINT "equipment_parentEquipmentId_fkey" FOREIGN KEY ("parentEquipmentId") REFERENCES "equipment"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
```

Both FKs `RESTRICT`: deleting a `Station` with existing equipment, or a dispenser with existing nozzles, fails at the DB level rather than cascading or (for the self-relation) silently nulling out a nozzle's parent.

## Documentation Updates

None needed. No API-spec or ERD doc exists yet in this repo (no `docs_root` configured, same as Tasks 01-04) — nothing else to update.

## Acceptance Criteria

1. `POST /equipment` (Dealer, `TANK`) creates a tank with `capacity`/`product`, returns `201`.
2. `POST /equipment` (Dealer, `DISPENSER`) creates a dispenser with no `capacity`/`product`/`currentMeterReading`, returns `201`.
3. `POST /equipment` (Dealer, `NOZZLE`) creates a nozzle attached to an existing dispenser with the supplied `currentMeterReading` baseline, returns `201`.
4. `POST /equipment` with a malformed body returns `400`.
5. `POST /equipment` with a `stationId` belonging to a different dealer returns `403`.
6. `POST /equipment` (`NOZZLE`) with a `parentEquipmentId` that isn't a `DISPENSER` at the same station returns `404`.
7. `POST /equipment` with an Admin token returns `403` (Admin cannot create equipment).
8. `GET /equipment` (Admin) returns all equipment across the network.
9. `GET /equipment` (Admin) with `?stationId=` returns only that station's equipment.
10. `GET /equipment` (Dealer) returns only their own station(s)' equipment (default, unfiltered).
11. `GET /equipment` (Dealer) with `?stationId=` set to their own station narrows to that station's equipment.
12. `GET /equipment` (Dealer) with `?stationId=` set to another dealer's station returns an empty array — the relation filter cannot be bypassed by the scalar filter.
13. `GET /equipment` for a Dealer-role caller with no linked `dealerId` returns `403`.
14. `GET /equipment/:id` returns `404` for a non-existent id (Admin caller).
15. `GET /equipment/:id` returns `403` for a Dealer requesting another dealer's equipment, via `dealerScope`.
16. `PATCH /equipment/:id` (Dealer, own `TANK`) updates `capacity`/`product`; `type`/`stationId`/`parentEquipmentId`/`currentMeterReading` remain unchanged even if included in the request body.
17. `PATCH /equipment/:id` (Dealer, own `DISPENSER`) with a `capacity` in the body leaves `capacity` as `null` (the type-scoped schema, not the tank one, applies).
18. `PATCH /equipment/:id` (Dealer, own `NOZZLE`) updates `identifier`; `currentMeterReading` remains unchanged even if included in the request body (the third type-scoped schema).
19. `PATCH /equipment/:id` returns `403` for a Dealer acting on another dealer's equipment.
20. `DELETE /equipment/:id` returns `403` for a Dealer acting on another dealer's equipment.
21. `PATCH` and `DELETE /equipment/:id` are both unreachable by an Admin token (`403` via `requireRole('DEALER')`).
22. `DELETE /equipment/:id` on a `DISPENSER` with an existing child `NOZZLE` returns `409`.
23. `DELETE /equipment/:id` on a nozzle, then its now-childless dispenser (in that order), both succeed with `200` and the deleted row's JSON.
24. `/equipment` routes reject an unauthenticated caller with `401` on both a collection route and a `:id` route.
25. The frontend Dealer station-equipment page renders tanks/dispensers/nozzles with their type visible.
26. The frontend Dealer station-equipment page's create forms (per type) and delete action call the correct `api/equipment.js` functions with the correct payload/id.
27. `MyStationsPage` renders a clickable "Equipment" link on each station card, pointing at that station's equipment page.
28. The frontend Admin equipment page renders equipment across stations with no create/edit/delete controls present, and `AdminHome` renders a clickable "Manage Equipment" link.
29. `npm run lint` succeeds with zero errors in both `backend/` and `frontend/`.
30. GitHub Actions CI runs lint + test for both backend and frontend green on the PR, with the live Postgres service.

## Branch

`feature/fuel_petroleum-XXX-equipment-management` (created from `story/3-dealer-fuel-station-management-system`; ticket number filled in at Phase 4i)

## Change Log

| Date | Time | Person | Change |
|------|------|--------|--------|
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Plan created. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | a_sag_plan_verifier independently reproduced the plan's central FK claim (both onDelete: Restrict live-DB tested, confirmed the SET NULL default too) and found 14 further issues, all fixed before approval: (1) adding a `<Link>` to `MyStationsPage` breaks its 3 existing bare-`render()` tests — added MemoryRouter wrapping to Files to Change. (2) `DealerShell.test.jsx` needed an `api/equipment` mock, same gap class as AdminShell's. (3) Decimal fields serialize as strings — test assertions now compare `Number(...)`. (4) Form-submitted numeric fields are strings; backend now uses `z.coerce.number()` rather than assuming the frontend sends real numbers. (5) Added a NOZZLE PATCH test — the third type-scoped schema was untested. (6) Added the orphan-dealer (no dealerId) 403 test for GET /equipment. (7) Added the Admin-blocked-on-POST test. (8) Added GET /equipment/:id 404-for-Admin and POST 400-validation tests. (9) Fixed backwards teardown reasoning: Restrict is *why* per-row delete order would matter, not why it doesn't — a single deleteMany is what makes order safe. (10) Pinned DELETE's response to 200+body (a bare 204 breaks the res.json() pattern every other api/*.js wrapper uses). (11) Removed a dead-code P2025 catch in the DELETE handler (unreachable given dealerScope already 403s first). (12) Split 4 compound acceptance criteria (old AC-6, AC-11, AC-12, AC-14) into one-behavior-per-row entries, growing the list from 17 to 27. (13) Renamed a misleading test description that implied an app-level cascade which doesn't exist. (14) Added "Manage Equipment" nav links to both AdminHome and the Dealer side so the new routes are actually reachable by clicking, not just by URL. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Re-verification pass confirmed all 14 fixes above landed correctly, and found 4 issues in the rewrite itself: (1) The Dealer-list-filter test/AC contradicted the plan's own Approach section — Approach says a foreign `stationId` filter yields zero rows (AND with the relation filter), but the Test Plan asserted "Dealer A's own equipment comes back anyway" (that's Task 04's `stations.js` pattern, which this endpoint deliberately does NOT copy). Split into three distinct ACs: default list, own-station filter, foreign-station filter → empty array. (2) `DealerShell.jsx`'s nav instruction contradicted itself (said to add a link to `DealerHome`, then said the link lives elsewhere) — resolved to a single design: no `DealerHome` change, the only click-path is the per-station link on `MyStationsPage`, now with its own test assertion. (3) `AdminEquipmentPage.test.jsx` didn't mock `api/stations.js` despite the page having a dealer→station picker — same unmocked-fetch-in-jsdom gap the plan itself calls out elsewhere. (4) **Real bug in fix #4 above:** `z.coerce.number()` alone silently turns an empty/unfilled form field into `0` instead of rejecting it — added `.positive()` (capacity) / `.nonnegative()` (currentMeterReading) bounds. Also added the two missing nav-clickability tests/ACs (Manage Equipment link on AdminHome, Equipment link on each station card) and fixed a wrong AC cross-reference (cited Task 04's AC-13 instead of AC-9). AC count grew from 27 to 30. |

## Execution Tracking

- **Started:** 2026-09-15
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-equipment-management
- **Collaborators:** (none yet)
