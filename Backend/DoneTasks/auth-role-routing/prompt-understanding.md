# Task 02: Auth & Role-Based Routing — Understanding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Depends on:** Task 01 (project scaffolding, PR #4, not yet merged into the story branch)
**Change Class:** FEATURE (new behavior — greenfield auth, nothing to preserve)

## What This Delivers

The shared login flow and the role-enforcement building blocks every later module (dealers, stations, orders, complaints, sales) will depend on. No dealer/station data model yet — this task only touches `User` and auth.

**Backend (Express + Prisma + Postgres, from `backend/`):**
- `POST /auth/login` — email/password, bcrypt-verified, issues a short-lived JWT access token (returned in the JSON body) and a long-lived refresh token (httpOnly, secure cookie).
- `POST /auth/refresh` — reads the refresh-token cookie, verifies it against its DB-stored hash, rotates it (issues + stores a new one, invalidates the old), returns a new access token.
- `POST /auth/logout` — clears the refresh-token cookie and invalidates the stored hash.
- Role-guard middleware: `requireRole('ADMIN' | 'DEALER')`, rejects with 403 if the JWT's role claim doesn't match.
- Dealer-scoping middleware: a reusable factory later modules (stations, orders, complaints, sales) call with a resource-ownership lookup function; verifies the resource's `dealerId` matches the authenticated Dealer's `dealerId` before letting the request through. Built now as infrastructure — no concrete resource to scope yet, since Dealer/Station tables don't exist until Tasks 03/04.
- `express-rate-limit` on `POST /auth/login` to blunt credential stuffing.
- `User` Prisma model: `id`, `email` (unique), `passwordHash`, `role` (`ADMIN`|`DEALER` enum), `dealerId` (nullable, **plain UUID column, no FK yet** — Task 03 introduces the `Dealer` table and a later migration adds the constraint), `refreshTokenHash` (nullable), timestamps.
- Seed script (`prisma/seed.js`, Prisma's standard seed hook) creates one Admin user and one Dealer user (dealerId left null — no Dealer record exists yet) so both login paths are provable.
- Request validation via `zod` (per spec's Error Handling section), returning `400` on schema failure.

**Frontend (React + Vite, from `frontend/`):**
- `react-router-dom` added — routes: `/login`, `/admin/*` (Admin shell), `/dealer/*` (Dealer shell), each shell an near-empty placeholder page behind its role guard.
- Login page (using the existing theme tokens/logo) posts credentials to `/auth/login`, stores the access token in memory (React context, not localStorage — reduces XSS exposure since the refresh token is already httpOnly), then redirects based on the role claim.
- A route guard component that reads the auth context and redirects unauthenticated or wrong-role users back to `/login`.
- An API client wrapper that attaches the access token as `Authorization: Bearer` and, on a 401, calls `/auth/refresh` once (cookie sent automatically) and retries the original request before giving up.

## Key Decisions (confirmed with user)

- **Refresh token:** httpOnly, secure cookie; a hash of it is stored in the DB against the user and rotated (replaced) on every use. Access token stays client-side in memory only, never persisted.
- **`User.dealerId`:** plain nullable UUID column in this task's migration, no foreign key. Task 03 owns creating the `Dealer` table; a later migration adds the FK. Keeps this task's migration additive and scoped to auth only, per the story's migration notes (`00_overview.md`).

## Out of Scope

- Any real Dealer/Station/Order/Complaint/Sales entity or business logic (Tasks 03+).
- Actual Admin/Dealer feature content — the two shells are placeholders proving the role split, not real dashboards.
- Self-registration — spec confirms dealers are created by Admin later (Task 03), not signed up. No `POST /auth/register` in this task.
- Audit logging (spec confirms not needed for MVP).

## Branching Strategy

Task 01's PR (#4) is open against the story branch but **not yet merged**, and this task needs its scaffolding (Express app, Prisma client, theme, CI). So:
- **Branch from:** `feature/fuel_petroleum-XXX-project-scaffolding` (has the scaffolding code)
- **Feature branch:** `feature/fuel_petroleum-XXX-auth-role-routing`
- **PR target:** `story/3-dealer-fuel-station-management-system` (not `main`, and not the scaffolding branch — same story-branch convention as Task 01)

## Applicable Rules

No project-specific coding standards installed yet (`standards_location` is unset in `.claude/config_hints.json`). Follow standard Express/Prisma/React/JWT community conventions, consistent with Task 01.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
