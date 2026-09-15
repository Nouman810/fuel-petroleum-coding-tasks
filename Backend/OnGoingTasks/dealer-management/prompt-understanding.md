# Task 03: Dealer Management — Understanding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Depends on:** Task 02 (auth & role-based routing, PR #5, merged)
**Change Class:** FEATURE (new Dealer entity + Admin CRUD; the login-suspension check extends existing auth behavior with a new rejection case rather than changing an existing contract)

## What This Delivers

Admin-only CRUD for the dealer network — the first real business entity in the system, replacing the placeholder Admin/Dealer shells with actual functionality.

**Backend (`backend/`):**
- `Dealer` Prisma model: `id`, `name`, `region`, `contactInfo`, `status` (`ACTIVE`|`SUSPENDED` enum), `createdAt`, `updatedAt`. Exact fields per spec.md's ERD (`spec.md` lines 65-71) — no invented fields.
- `User.dealerId` gets a real foreign key to `Dealer.id` in this task's migration. Task 02 deliberately left it as a plain UUID column with no FK since no `Dealer` table existed yet; now that it does, the FK is added immediately rather than left dangling — there's no reason to defer it further once the referenced table exists.
- `POST /dealers` (Admin only) — creates a `Dealer` row and its linked `User` row (role `DEALER`, `dealerId` set to the new dealer's id) in one transaction. Request includes the dealer's `name`, `region`, `contactInfo`, plus `email` and `password` for the linked login account (Admin sets the password directly — confirmed with user, no email/notification service exists yet in this project to support an auto-generated-password flow).
- `GET /dealers` (Admin only) — list all dealers, response includes `region` and `status` for at-a-glance display.
- `GET /dealers/:id` (Admin only) — single dealer detail.
- `PATCH /dealers/:id` (Admin only) — edit `name`/`region`/`contactInfo`.
- `PATCH /dealers/:id/status` (Admin only) — set `status` to `ACTIVE` or `SUSPENDED`.
- Login flow change: `POST /auth/login` currently authenticates purely on email+password (`backend/src/routes/auth.js`, the `prisma.user.findUnique` + `comparePassword` block). Add a check, after password verification succeeds and only for `role === 'DEALER'` users, that the linked `Dealer.status` is `ACTIVE`; reject with `401` (same "Invalid credentials" family as a wrong password, so the check plugs into the existing failure path rather than becoming a distinct new error surface) if the dealer is suspended. This means the login handler needs to load the user's `Dealer` row (via the new FK) when the role is `DEALER`.
- Request validation via `zod` (matching Task 02's pattern), `400` on schema failure. `404` if a dealer id doesn't exist. `409` if `email` is already taken when creating a dealer's linked user (same unique-constraint conflict class as the spec's Error Handling section describes).
- Everything under Admin-only guard (`requireRole('ADMIN')` from Task 02's `backend/src/middleware/auth.js`).

**Frontend (`frontend/`):**
- Replaces the placeholder `AdminShell.jsx` heading with real dealer-management routes nested under `/admin`: a dealer list page and a dealer detail/edit page, plus a create-dealer form.
- List page: table of dealers showing name, region, status at a glance, link to each dealer's detail page, and a way to open the create form.
- Detail page: shows the dealer's fields, an edit form (name/region/contactInfo), and a status toggle (activate/suspend).
- Create form: name, region, contactInfo, email, password for the linked login account.
- All calls go through the existing `apiFetch` client (`frontend/src/api/client.js`) from Task 02, which already attaches the bearer token and retries once after a 401 refresh.

## Out of Scope

- Dealer-facing UI — a Dealer can't yet do anything with their own profile beyond logging in (confirmed by raw_prompt.md). No `/dealer/*` routes change in this task.
- Stations, orders, or any dealer-scoped resource — those tables don't exist until Tasks 04/07. The spec's "view a dealer's stations and order history" (spec.md line 164) is deferred to whichever task adds reporting on top of those tables; there's nothing to view yet.
- Dealer self-service password reset/change — no such flow exists anywhere in the app yet.
- Audit logging — spec confirms not needed for MVP; `createdAt`/`updatedAt` on `Dealer` is the extent of traceability for this task.

## Key Decisions (confirmed with user)

- **Initial dealer password:** Admin sets it directly in the create-dealer request (not auto-generated). Simplest option given no email/notification infrastructure exists yet to deliver a generated password to the dealer out-of-band.
- **`User.dealerId` FK:** added now, in this task's migration, since the `Dealer` table now exists and there's no reason to leave the column unconstrained once it does.

## Applicable Rules

`standards_location` is unset in `.claude/config_hints.json` (no project-specific coding standards installed). Follow standard Express/Prisma/React/zod conventions, consistent with Tasks 01 and 02.

## Branching Strategy

This task is part of a multi-PR story. Branch from and target PRs to the story branch:
- **Story branch:** story/3-dealer-fuel-station-management-system
- **Feature branch:** feature/fuel_petroleum-XXX-dealer-management (branched from story/3-dealer-fuel-station-management-system, ticket-late — same convention as Tasks 01 and 02, ticket number filled in at Phase 4i)
- **PR target:** story/3-dealer-fuel-station-management-system (not main)

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
