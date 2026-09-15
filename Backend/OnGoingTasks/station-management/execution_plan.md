# Execution Plan: Task 04 — Station Management

**Branch:** `feature/fuel_petroleum-XXX-station-management` (branched from `story/3-dealer-fuel-station-management-system`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system`
**Change Class:** FEATURE

## Summary

Add the `Station` entity: Admin creates a station and assigns it to a dealer, views/filters the full network; a Dealer views and edits only their own station(s)' `address`/`contactInfo` via the existing `dealerScope` middleware. Replaces the `DealerShell` placeholder with a real "My Stations" page.

## Approach & Trade-offs

- **`Station.dealerId` is a required FK** (`String`, not `String?`), unlike `User.dealerId`. A station always belongs to exactly one dealer per the spec's ERD (no "nullable" annotation, unlike `User.dealerId`). Confirmed via `prisma migrate diff` against the real schema: Prisma's default referential action for a required relation is `ON DELETE RESTRICT ON UPDATE CASCADE` — deleting a `Dealer` row (not exposed by any endpoint in this task or Task 03) would be blocked while it still owns stations, rather than silently orphaning or cascade-deleting them.
- **`contactInfo` added to `Station`**, confirmed with the user in Phase 1 — the spec's ERD table doesn't list it, but the raw prompt explicitly calls for a Dealer to edit "address and contact details."
- **The Dealer-facing list (`GET /stations`) is scoped by a plain query filter (`where: { dealerId: req.user.dealerId }`), not `dealerScope`.** `dealerScope` (from Task 02) is built to check ownership of a *single* resource, resolved via a `getOwnerDealerId(req)` callback — it has no concept of filtering a list. For the list endpoint, forcing the `where` clause to the caller's own `dealerId` when `req.user.role === 'DEALER'` is the correct primitive; `dealerScope` is reserved for `GET/PATCH /stations/:id`, where it directly satisfies the raw prompt's instruction to reuse it "not a separate ad-hoc check."
- **Explicit guard against `req.user.dealerId` being unset for a Dealer-role caller.** `a_sag_plan_verifier` caught that Prisma silently *drops* a `where` key whose value is `undefined` — `where: { dealerId: undefined }` compiles to no filter at all (confirmed empirically), not "match nothing." `User.dealerId` is optional (a `DEALER`-role user isn't guaranteed to be linked to a `Dealer` yet, and the seeded test dealer from Task 02 has `dealerId: null`), so this is a real, representable state, not a hypothetical. The handler therefore checks `if (!req.user.dealerId) return res.status(403).json(...)` before building the `where` clause, rather than relying on `null` (which correctly becomes `IS NULL`) always being what's present — the login flow happens to load a full `null`, but nothing enforces that invariant at this layer, so the route guards for itself.
- **No Admin edit/reassign endpoint in this task.** The raw prompt asks for Admin to create + assign + list/filter; it never asks for post-creation admin edits or reassignment. Building that now would be scope beyond what was asked — a future task can add it if a real need shows up.
- **Dealer's `PATCH /stations/:id` schema only accepts `address` and `contactInfo`.** Even if a Dealer's request body includes `name`/`region`/`dealerId`, zod's default behavior strips unrecognized keys before the Prisma update runs — the same pattern already relied on on `PATCH /dealers/:id` (Task 03). This is the enforcement mechanism for "not reassignment," not just a documentation note.
- **`vite.config.js`'s dev proxy gets a `/stations` entry.** `a_sag_plan_verifier` caught this exact gap for `/dealers` in Task 03's plan; adding it up front here rather than repeating that discovery.
- **Frontend: no separate "station detail" route.** The Admin side only needs list + filter + create (no edit asked for). The Dealer side needs to handle zero/one/many stations per the raw prompt, so "My Stations" renders each station as its own editable card on one page rather than needing a per-station URL.

## Files to Change

**Backend** (`backend/`)
- `prisma/schema.prisma` — add `Station` model; add `stations Station[]` to `Dealer`. See Database Schema below.
- `prisma/migrations/20260915130000_add_station/migration.sql` — new, generated via `npx prisma migrate diff --from-schema-datamodel <current> --to-schema-datamodel <draft> --script` (real tool output, see Database Schema). Timestamped after Task 03's `20260915120000_add_dealer`.
- `src/routes/stations.js` — new:
  - `router.use('/stations', authenticate)` — every route needs a valid session; role-specific access is enforced per-route below (not a blanket `requireRole('ADMIN')`, since Dealers legitimately call `GET /stations`, `GET /stations/:id`, `PATCH /stations/:id`).
  - `POST /stations` — `requireRole('ADMIN')`, zod-validated (`name`, `address`, `region`, `contactInfo`, `dealerId`), `404` if `dealerId` doesn't reference an existing dealer (checked via `prisma.dealer.findUnique` before create — a bad FK reference must not surface as an opaque 500), `201` on success.
  - `GET /stations` — Admin: `prisma.station.findMany` with optional `dealerId`/`region` `where` filters from query params. Dealer: `403` if `req.user.dealerId` is falsy (see Approach); otherwise `where` is built fresh as `{ dealerId: req.user.dealerId }` only — any `dealerId`/`region` query params the caller sent are ignored entirely (a Dealer has no reason to filter across stations they can't see anyway).
  - `GET /stations/:id` — wrapped in `dealerScope(async (req) => (await prisma.station.findUnique({ where: { id: req.params.id } }))?.dealerId)`. Admin bypasses (per `dealerScope`'s existing behavior). Dealer: 403 if the station isn't theirs (or doesn't exist — not confirming/denying existence to a non-owner, consistent with `/dealers`). Admin path checks existence itself and returns `404` if not found.
  - `PATCH /stations/:id` — same `dealerScope` wrapping, zod schema restricted to `{ address, contactInfo }` (see Approach). `404` if the station doesn't exist (Admin never reaches this path since there's no Admin-facing edit UI, but the route itself doesn't discriminate by role beyond `dealerScope`'s own Admin-bypass — an Admin bearer token could technically call it too, which is fine, it's the same ownership-neutral behavior `dealerScope` already gives Admin on `/dealers`).
- `src/routes/stations.test.js` — new, see Test Plan.
- `src/app.js` — mount `stationsRouter` (`require('./routes/stations')`, `app.use(stationsRouter)`), same pattern as `dealersRouter`.

**Frontend** (`frontend/`)
- `vite.config.js` — add `'/stations': 'http://localhost:3000'` to `server.proxy`.
- `src/api/stations.js` — new: `listStations(params)` (accepts `{ dealerId, region }`, builds a query string), `getStation(id)`, `createStation(payload)`, `updateStation(id, payload)`. Same `apiFetch`-wrapping pattern as `api/dealers.js`.
- `src/routes/AdminShell.jsx` — add a `stations` route (`StationListPage`) and a "Manage Stations" link on the Admin home.
- `src/routes/AdminShell.test.jsx` — extend: add `vi.mock('../api/stations', () => ({ listStations: vi.fn(), getStation: vi.fn(), createStation: vi.fn(), updateStation: vi.fn() }))` alongside the existing `api/dealers` mock (the file's current `vi.mock('../api/dealers', ...)` factory only covers `api/dealers`; once `AdminShell` renders `StationListPage`, which imports `api/stations`, that module would otherwise hit the real `apiFetch`/jsdom `fetch`). New case: stub both `listStations` and `listDealers` (the latter feeds `StationListPage`'s dealer-picker `<select>` — it must not resolve `undefined`) and assert `/admin/stations` resolves to `StationListPage`'s content.
- `src/routes/StationListPage.jsx` — new: fetches and renders all stations (name, dealer, region) with `dealerId`/`region` filter controls (dealer filter is a `<select>` populated via `listDealers()` from `api/dealers.js`, re-triggers `listStations` on change); a toggleable "New Station" form (name, address, region, contactInfo, and a dealer `<select>`) that calls `createStation`.
- `src/routes/StationListPage.test.jsx` — new, see Test Plan.
- `src/routes/DealerShell.jsx` — rewritten: replaces the static placeholder with its own nested `<Routes>` (`index` → a small nav, `stations` → `MyStationsPage`), mirroring `AdminShell`'s structure from Task 03.
- `src/routes/DealerShell.test.jsx` — new (no prior test existed for the placeholder), asserts `/dealer` and `/dealer/stations` resolve to the right content via a real `MemoryRouter` matching `App.jsx`'s `/dealer/*` delegation.
- `src/routes/MyStationsPage.jsx` — new: fetches the caller's own stations via `listStations()` (no params — the backend already scopes it), renders each as its own card with an editable `address`/`contactInfo` form and its own Save button; renders a clear empty state when the list has zero stations (per the raw prompt: a dealer can have zero, one, or multiple).
- `src/routes/MyStationsPage.test.jsx` — new, see Test Plan.

**Root**
- No `.github/workflows/ci.yml` changes needed — no new env vars, no new services.
- No `README.md` changes needed — no new local-setup steps.

## Test Plan

**Backend** (`stations.test.js`) — creates its own fixture Admin (for auth) and two fixture Dealers with linked `DEALER` users/tokens against the live Postgres. **Teardown order matters**: `station.deleteMany` → `user.deleteMany` → `dealer.deleteMany` in `afterAll` — the FK is `ON DELETE RESTRICT` (see Database Schema), so deleting a `Dealer` while it still owns a station throws `P2003` and leaves the DB dirty for the next run; `dealers.test.js`'s teardown order (users then dealers) has no stations in play and cannot be copied as-is.
- `POST /stations` (Admin) with a valid `dealerId` → `201`, response has the station's fields and correct `dealerId`
- `POST /stations` with a non-existent `dealerId` → `404`
- `GET /stations` (Admin) → includes the created station; `?dealerId=` filters to just that dealer's stations; `?region=` filters by region
- `GET /stations` (Dealer A's token) → only Dealer A's station(s), even when the request includes a `?dealerId=<Dealer B's id>` query param, AND even when it includes a `?region=<a region Dealer A's station isn't in>` query param — both filter params must be proven ignored, not just `dealerId` (a plausible wrong implementation overrides only `dealerId` while still honoring `region` for a Dealer)
- `GET /stations/:id` (Admin) for a non-existent id → `404`
- `GET /stations/:id` (Dealer A's token) for Dealer A's own station → `200`
- `GET /stations/:id` (Dealer B's token) for Dealer A's station → `403`
- `PATCH /stations/:id` (Dealer A's token) on their own station, body includes `address`, `contactInfo`, AND `name`/`region` → `200`, `address`/`contactInfo` updated, `name`/`region` unchanged from creation (proves the schema strips them, not just that valid fields work)
- `PATCH /stations/:id` (Dealer B's token) on Dealer A's station → `403`
- `PATCH /stations/:id` (Admin's token) for a non-existent station id → `404` (the only caller that can actually reach this branch — `dealerScope` 403s a Dealer before the handler's own not-found check runs)
- `GET /stations` with no `Authorization` header → `401` (the shared router-level `authenticate` guard); `GET /stations/:id` with no header → `401` too, proving the guard covers the `:id` route as well as the collection route, not just one path shape

**Frontend:**
- `StationListPage.test.jsx` — mocks `api/stations.js` and `api/dealers.js`: renders a table row per station with name/dealer/region visible; changing the dealer filter calls `listStations` with the selected `dealerId`; submitting the "New Station" form calls `createStation` with the entered values including the selected `dealerId`
- `MyStationsPage.test.jsx` — mocks `api/stations.js`: renders one card per returned station; renders an explicit empty-state message when the list is empty; editing and saving one station's form calls `updateStation(id, { address, contactInfo })`
- `AdminShell.test.jsx` — new case: `/admin/stations` resolves to `StationListPage`'s content
- `DealerShell.test.jsx` — `/dealer` resolves to the Dealer home nav, `/dealer/stations` resolves to `MyStationsPage`'s content

**CI:** both suites run via the existing `.github/workflows/ci.yml` jobs, no changes needed there. Additionally, this dev machine now has a local Postgres available (set up during Task 03/login verification work) — the backend suite will be run and verified locally in Phase 3/4 as well, not only via CI.

## Database Schema

Added to `prisma/schema.prisma`:

```prisma
model Station {
  id          String   @id @default(uuid())
  dealerId    String
  dealer      Dealer   @relation(fields: [dealerId], references: [id])
  name        String
  address     String
  region      String
  contactInfo String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("stations")
}
```

`Dealer` gains the inverse relation:
```prisma
model Dealer {
  id          String       @id @default(uuid())
  name        String
  region      String
  contactInfo String
  status      DealerStatus @default(ACTIVE)
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
  users       User[]
  stations    Station[]

  @@map("dealers")
}
```

Migration SQL (generated via `npx prisma migrate diff --from-schema-datamodel ./prisma/schema.prisma --to-schema-datamodel <draft-schema-with-the-above> --script`, run against the current committed schema — no live DB connection needed for this command; real tool output):

```sql
-- CreateTable
CREATE TABLE "stations" (
    "id" TEXT NOT NULL,
    "dealerId" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "address" TEXT NOT NULL,
    "region" TEXT NOT NULL,
    "contactInfo" TEXT NOT NULL,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "stations_pkey" PRIMARY KEY ("id")
);

-- AddForeignKey
ALTER TABLE "stations" ADD CONSTRAINT "stations_dealerId_fkey" FOREIGN KEY ("dealerId") REFERENCES "dealers"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
```

`ON DELETE RESTRICT` is Prisma's default for a required relation (confirmed by running the diff tool, not assumed) — deleting a `Dealer` that still owns stations would fail at the DB level rather than silently cascading or orphaning. No endpoint in this task or Task 03 deletes a `Dealer`, so this is a safety backstop, not exercised behavior yet.

## Documentation Updates

None needed. No API-spec or ERD doc exists yet in this repo (no `docs_root` configured, same as Tasks 01-03) — nothing else to update.

## Acceptance Criteria

1. `POST /stations` (Admin) creates a station assigned to an existing dealer, returns `201`.
2. `POST /stations` with a non-existent `dealerId` returns `404`.
3. `GET /stations` (Admin) returns all stations and supports `?dealerId=` and `?region=` filters.
4. `GET /stations` (Dealer) returns only that dealer's own station(s) — zero, one, or many — regardless of any filter query params sent.
5. `GET /stations/:id` (Admin) returns any station, `404` for a non-existent id.
6. `GET /stations/:id` (Dealer) returns their own station; `403` for another dealer's station, via the `dealerScope` middleware.
7. `PATCH /stations/:id` (Dealer, own station) updates `address`/`contactInfo`; `name`/`region` remain unchanged even if included in the request body.
8. `PATCH /stations/:id` (Dealer) on another dealer's station returns `403`, via `dealerScope`.
9. The `/stations` router's shared `authenticate` guard rejects an unauthenticated caller with `401` on both a collection route (`GET /stations`) and a `:id` route (`GET /stations/:id`).
10. `PATCH /stations/:id` for a non-existent station id returns `404` (Admin caller — the only path that reaches this branch, since `dealerScope` 403s a Dealer first).
11. The frontend Admin stations page renders all stations with dealer/region filters, and creating a station via its form (including dealer assignment) succeeds.
12. The frontend Dealer "My Stations" page renders the dealer's own station(s) — including a clear empty state for zero stations — and editing `address`/`contactInfo` succeeds.
13. `npm run lint` succeeds with zero errors in both `backend/` and `frontend/`.
14. GitHub Actions CI runs lint + test for both backend and frontend green on the PR, with the live Postgres service.

## Branch

`feature/fuel_petroleum-XXX-station-management` (created from `story/3-dealer-fuel-station-management-system`; ticket number filled in at Phase 4i)

## Change Log

| Date | Time | Person | Change |
|------|------|--------|--------|
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Plan created. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | a_sag_plan_verifier found 5 issues, all fixed before user review: (1) **Data-leak edge case:** `where.dealerId = req.user.dealerId` silently disabled the Dealer-scope filter entirely when `dealerId` was `undefined` (Prisma drops `undefined` where-keys) — a representable state since `User.dealerId` is optional. Added an explicit `403` guard when `req.user.dealerId` is falsy, rather than relying on `null` always being what's present. (2) AC-4's "`regardless of any filter query params`" only had a `dealerId` test case, not `region` — added a `?region=` case, since a plausible wrong implementation would only override `dealerId`. (3) The planned `afterAll` teardown order (mirroring `dealers.test.js`) would throw `P2003` given the `ON DELETE RESTRICT` FK — specified `station → user → dealer` deletion order explicitly. (4) Extending `AdminShell.test.jsx` needs a new `vi.mock('../api/stations')` (the file's existing mock only covers `api/dealers`) — added to Files to Change. (5) AC-9 claimed all `/stations` routes reject unauthenticated callers but only tested one route shape, and the `PATCH .../404` branch (Admin-only, since `dealerScope` 403s a Dealer first) had no AC or test — narrowed AC-9's wording and split the 404 case into its own AC-10. |

## Execution Tracking

- **Started:** 2026-09-15
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-station-management
- **Collaborators:** (none yet)
