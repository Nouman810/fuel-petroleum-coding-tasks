# Task 04: Station Management — Understanding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Depends on:** Task 03 (dealer management, PR #6, merged)
**Change Class:** FEATURE (new Station entity + CRUD; nothing existing changes contract)

## What This Delivers

The `Station` entity — the piece equipment, orders, complaints, and sales all hang off of in later tasks. Admin creates and assigns stations to dealers; a Dealer manages their own station(s)' profile only.

**Backend (`backend/`):**
- `Station` Prisma model: `id`, `dealerId` (required FK to `Dealer.id` — a station always belongs to exactly one dealer, unlike `User.dealerId` which is optional), `name`, `address`, `region`, `contactInfo`, `createdAt`, `updatedAt`. Fields per spec.md's ERD (`spec.md` lines 72-78) plus a `contactInfo` field (confirmed with user — the ERD doesn't list one, but the raw prompt explicitly calls for a Dealer to edit "address and contact details," and a station-level contact is genuinely useful once orders/complaints route to a specific station).
- `Dealer` model gains the inverse relation: `stations Station[]`.
- `POST /stations` (Admin only) — creates a station and assigns it to a dealer in one request (`name`, `address`, `region`, `contactInfo`, `dealerId`). `404` if `dealerId` doesn't reference an existing dealer.
- `GET /stations` — Admin: lists all stations, with optional `?dealerId=` and `?region=` query filters. Dealer: automatically scoped to their own stations only (ignores any filter query params — a Dealer has no reason to filter across a network they can't see), returns a list (zero, one, or many — never assume exactly one, per the raw prompt).
- `GET /stations/:id` — Admin: any station, `404` if not found. Dealer: only their own, via the `dealerScope` middleware from Task 02 (`backend/src/middleware/auth.js`) — not a separate ad-hoc ownership check, per the raw prompt's explicit instruction. A Dealer requesting another dealer's station id gets `403` (ownership is not confirmed or denied by existence, consistent with the "never sufficient to read/write another dealer's data" principle already established for `/dealers`).
- `PATCH /stations/:id` (Dealer only, own station, via `dealerScope`) — updates `address` and `contactInfo` only. `name`, `region`, and `dealerId` are NOT editable here — the raw prompt is explicit that a Dealer edits "address and contact details, not reassignment," and by extension not the network-classification fields (`name`, `region`) that Admin sets at creation time.
- No Admin edit/reassign endpoint in this task — the raw prompt only asks for Admin to create + assign + list/filter + (implicitly) view; it never asks for post-creation admin edits or station reassignment. Not building that now avoids scope creep beyond what was asked; a future task can add it if needed.
- Validation via `zod` (matching the established pattern), `400` on schema failure, `404` for a nonexistent dealer on create or a nonexistent station for Admin's `GET /:id`.
- Everything Admin-only routes guarded by `requireRole('ADMIN')`; the Dealer-facing routes (`GET /stations` list, `GET /stations/:id`, `PATCH /stations/:id`) guarded by `authenticate` and reachable by either role, with the actual scoping enforced per-route as described above (list: filtered in the query; single-resource: via `dealerScope`).

**Frontend (`frontend/`):**
- `AdminShell` gains a stations section: a list page (table of all stations with dealer/region filter controls) and a create-station form (assign to a dealer via a dropdown/select populated from `GET /dealers`).
- `DealerShell` — currently a static placeholder heading (`frontend/src/routes/DealerShell.jsx`) — gets rewritten with a real "My Stations" page: lists the dealer's own stations (handles zero/one/many, per the raw prompt), each with an inline or per-station edit form for `address`/`contactInfo`.
- All calls go through the existing `apiFetch` client (`frontend/src/api/client.js`).
- `vite.config.js`'s dev proxy needs a `/stations` entry alongside the existing `/auth` and `/dealers` ones (`frontend/vite.config.js`), following the exact gap `a_sag_plan_verifier` caught in Task 03's plan for `/dealers`.

## Out of Scope

- Equipment, orders, complaints, sales — those tables don't exist until later tasks; nothing here references them.
- Station reassignment or Admin post-creation edits (see above).
- A Dealer creating their own stations — only Admin creates stations, per the raw prompt.

## Key Decisions (confirmed with user)

- **`contactInfo` added to `Station`**, not in the spec's ERD table but confirmed necessary to satisfy the raw prompt's explicit "address and contact details" wording for Dealer edits.

## Applicable Rules

`standards_location` is unset in `.claude/config_hints.json` (no project-specific coding standards installed). Follow the same Express/Prisma/React/zod conventions established in Tasks 01-03.

## Branching Strategy

This task is part of a multi-PR story. Branch from and target PRs to the story branch:
- **Story branch:** story/3-dealer-fuel-station-management-system
- **Feature branch:** feature/fuel_petroleum-XXX-station-management (branched from story/3-dealer-fuel-station-management-system, ticket-late — same convention as Tasks 01-03, ticket number filled in at Phase 4i)
- **PR target:** story/3-dealer-fuel-station-management-system (not main)

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
