# Execution Plan: Task 03 — Dealer Management

**Branch:** `feature/fuel_petroleum-XXX-dealer-management` (branched from `story/3-dealer-fuel-station-management-system`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system`
**Change Class:** FEATURE

## Summary

Add the `Dealer` entity and Admin-only CRUD for it — the first real business entity in the system. Creating a dealer also creates its linked login `User` (role `DEALER`). A suspended dealer's login and any subsequent token refresh are both rejected, even with correct credentials or a still-valid refresh cookie. Replaces the placeholder `AdminShell` with a real dealer list/detail/create/edit UI.

## Approach & Trade-offs

- **`User.dealerId` gets a real FK to `Dealer.id` in this task's migration.** Task 02 deliberately left it as a plain nullable UUID column since no `Dealer` table existed yet. That table exists now, so the FK is added immediately — there's no reason to leave the column unconstrained once its target exists.
- **Dealer creation is atomic (`prisma.$transaction`).** A dealer without a login account, or a `User` with a `dealerId` pointing at nothing, are both broken states. Matches the spec's Error Handling note that multi-table mutations run inside a transaction.
- **Suspension is checked in *both* `POST /auth/login` and `POST /auth/refresh`, not just login.** A first pass at this plan checked only login; `a_sag_plan_verifier` caught that a dealer suspended mid-session keeps a working access token for up to 7 days (`REFRESH_TOKEN_TTL`), since `/auth/refresh` re-issues tokens from `prisma.user.findUnique({ where: { id: payload.sub } })` with no dealer check, and `AuthContext` silently restores that session on every page load. Both handlers now use `include: { dealer: true }` on their user lookup and apply the same `user.role === 'DEALER' && user.dealer?.status === 'SUSPENDED'` gate. This closes the stale-session window instead of leaving it as a known gap.
- **The check is written as `user.dealer?.status === 'SUSPENDED'`, not a truthy/falsy check on `user.dealer`.** The existing seeded `DEALER` test user (`prisma/seed.js`) has `dealerId: null`, so `include: { dealer: true }` returns `dealer: null` for it — `null?.status` short-circuits to `undefined`, which is not `=== 'SUSPENDED'`, so the check no-ops rather than crashing or wrongly rejecting a dealer account with no linked `Dealer` row.
- **Suspended-dealer rejection returns `401` with the same `{ error: 'Invalid credentials' }` (login) / `{ error: 'Invalid refresh token' }` (refresh) bodies already used for those endpoints' existing failure cases** — not a distinct error code. Matches the raw prompt's framing ("rejected even with correct credentials") and avoids leaking to an unauthenticated caller *why* a request failed. For `/auth/refresh`, this also means the refresh-token cookie and stored hash are cleared exactly as they already are for an invalid/reused token, so a suspended dealer's browser doesn't keep silently retrying.
- **Create-dealer takes `email` + `password` directly in the request body** (confirmed with user in Phase 1) — the Admin sets the dealer's initial password themselves. No email/notification service exists in this project to support an auto-generate-and-deliver flow.
- **`PATCH /dealers/:id` and `PATCH /dealers/:id/status` are separate endpoints.** Editing profile fields and toggling status are different operations with different callers in mind on the frontend (an edit form vs. a single status-toggle button); splitting them keeps each request body single-purpose.
- **The `/dealers` auth guard is scoped with a path argument: `router.use('/dealers', authenticate, requireRole('ADMIN'))`.** A first pass at this plan used a path-less `router.use(authenticate, requireRole('ADMIN'))`, which `a_sag_plan_verifier` flagged: a path-less `router.use()` matches `/` and therefore runs for *every* request that reaches this router, not just `/dealers*`. Since this router is mounted after `healthRouter`/`authRouter` in `app.js`, any request neither of those handled would fall through into this guard and get a `401` instead of Express's normal 404. Scoping the guard to `/dealers` keeps this router's routes as the only ones affected, consistent with `health.js`/`auth.js` declaring full paths.
- **Frontend: `AdminShell` gains its own nested `<Routes>`** (`/admin` index → a small nav, `/admin/dealers` → list, `/admin/dealers/:id` → detail). `App.jsx`'s existing `/admin/*` route already delegates the whole subtree to `AdminShell`, so nesting routes inside it is the natural extension.
- **`vite.config.js`'s dev proxy gets a `/dealers` entry.** Only `/auth` is proxied today. Without adding `/dealers`, a relative `apiFetch('/dealers')` in dev would hit the Vite dev server itself, which returns the SPA's `index.html` with a `200` — `res.ok` would be true and `res.json()` would throw a parse error instead of failing cleanly. This gap wouldn't show up in the frontend test suite (both new page tests mock `api/dealers.js`), only in a real `npm run dev` session, which is exactly why it's called out explicitly here rather than left implicit.
- **A thin `frontend/src/api/dealers.js` wrapper** (following `api/client.js`'s `apiFetch` usage pattern from Task 02, which returns the raw `Response` and does not throw) keeps the page components free of fetch/error-shape boilerplate.

## Files to Change

**Backend** (`backend/`)
- `prisma/schema.prisma` — add `DealerStatus` enum and `Dealer` model; add `dealer Dealer? @relation(fields: [dealerId], references: [id])` to `User`. See Database Schema below.
- `prisma/migrations/20260915120000_add_dealer/migration.sql` — new, generated via `npx prisma migrate diff --from-schema-datamodel <current> --to-schema-datamodel <draft> --script` (real tool output, see Database Schema section). Timestamped after the existing `20260915000000_add_user_auth` so migrations still sort correctly (both land on 2026-09-15).
- `src/routes/dealers.js` — new. `router.use('/dealers', authenticate, requireRole('ADMIN'))` (path-scoped — see Approach), then:
  - `POST /dealers` — zod-validated (`name`, `region`, `contactInfo`, `email`, `password`), creates `Dealer` + `User` in one transaction, `201`. Catches Prisma `P2002` (unique `email` violation) → `409`.
  - `GET /dealers` — list all, `200`.
  - `GET /dealers/:id` — `200`, or `404` if not found.
  - `PATCH /dealers/:id` — zod-validated partial update (`name`/`region`/`contactInfo`, all optional), `200`, or `404`.
  - `PATCH /dealers/:id/status` — zod-validated (`status: 'ACTIVE'|'SUSPENDED'`), `200`, or `404`.
- `src/routes/dealers.test.js` — new, see Test Plan.
- `src/routes/auth.js` — modify both handlers:
  - `POST /auth/login`: change `prisma.user.findUnique({ where: { email } })` to `prisma.user.findUnique({ where: { email }, include: { dealer: true } })`; after the existing password check, add the suspension gate (see Approach), `401` with the existing `{ error: 'Invalid credentials' }` body.
  - `POST /auth/refresh`: change `prisma.user.findUnique({ where: { id: payload.sub } })` to include `{ dealer: true }`; after the existing refresh-token-hash verification succeeds (and before `issueTokensForUser`), add the same suspension gate, clearing the cookie and stored hash exactly as the existing invalid-token branch does, `401` with `{ error: 'Invalid refresh token' }`.
- `src/routes/auth.test.js` — modify:
  - Add a **new, separate** `describe('POST /auth/login and /auth/refresh with a suspended dealer', ...)` block (its own `jest.resetModules()` + raised `LOGIN_RATE_LIMIT_MAX`, same pattern as the existing rate-limit `describe`) rather than adding cases to the main `describe('auth routes', ...)` block — that block already performs 3 logins against the shared in-memory rate limiter, and CI's `LOGIN_RATE_LIMIT_MAX` is 5, so two more logins in the same block would land exactly on the limit with zero headroom for future tests.
  - This new block gets **its own** `afterAll` that deletes its fixture `User` rows (by email) and then its `Dealer` rows. The existing `describe('auth routes', ...)`'s `afterAll` (lines 27-30) is scoped to that describe and runs before this new, later-declared describe's tests even execute — extending it would not clean up fixtures that don't exist yet at that point in the run. Each describe block owns and cleans up only its own fixtures.
- `src/app.js` — mount `dealersRouter` (`require('./routes/dealers')`, `app.use(dealersRouter)`), same pattern as `authRouter`. Safe now that the guard inside `dealers.js` is path-scoped.

**Frontend** (`frontend/`)
- `vite.config.js` — add `'/dealers': 'http://localhost:3000'` to `server.proxy` (see Approach).
- `src/api/dealers.js` — new: `listDealers()`, `getDealer(id)`, `createDealer(payload)`, `updateDealer(id, payload)`, `updateDealerStatus(id, status)`. Each wraps `apiFetch` from `api/client.js`, throws on a non-ok response with the server's `error` message when present.
- `src/routes/AdminShell.jsx` — rewritten: renders its own `<Routes>` (`index` → a small nav linking to Dealers, `dealers` → `DealerListPage`, `dealers/:id` → `DealerDetailPage`). Replaces the static placeholder heading.
- `src/routes/AdminShell.test.jsx` — new, see Test Plan. Added because the routing wiring itself (does `/admin/dealers` actually resolve to the list page, does `/admin/dealers/:id` resolve to the detail page) is otherwise unverified — the page-level tests below render each page directly, not through `AdminShell`'s routes.
- `src/routes/DealerListPage.jsx` — new: fetches and renders a table (name, region, status) with a link to each dealer's detail page; a toggleable "New Dealer" form (name, region, contactInfo, email, password) that calls `createDealer` and refreshes the list on success.
- `src/routes/DealerListPage.test.jsx` — new, see Test Plan.
- `src/routes/DealerDetailPage.jsx` — new: fetches one dealer by the `:id` route param, renders an edit form (name/region/contactInfo) that calls `updateDealer`, and a status toggle button that calls `updateDealerStatus`.
- `src/routes/DealerDetailPage.test.jsx` — new, see Test Plan.

**Root**
- No `.github/workflows/ci.yml` changes needed — no new env vars, no new services.
- No `README.md` changes needed — no new local-setup steps.

## Test Plan

**Backend:**
- `dealers.test.js` — creates its own fixture Admin (for auth) against the live Postgres, cleans up in `afterAll`:
  - `POST /dealers` with valid data → `201`, response has the dealer's `id`/`name`/`region`/`contactInfo`/`status: 'ACTIVE'`, and a `User` row was created with `role: 'DEALER'` and `dealerId` equal to the new dealer's id (verified via a direct Prisma query in the test)
  - `POST /dealers` with an email already in use → `409`
  - `GET /dealers` → `200`, includes the created dealer with `region` and `status` present
  - `GET /dealers/:id` for a non-existent id → `404`
  - `PATCH /dealers/:id` updates `name`/`region`/`contactInfo` → `200`, changed fields reflected
  - `PATCH /dealers/:id/status` with `{ status: 'SUSPENDED' }` → `200`, `status` is `SUSPENDED`
  - Any `/dealers` route called without a valid Admin bearer token → `401`; called with a Dealer-role token → `403`
- `auth.test.js` — new isolated `describe` block (own `jest.resetModules()` + raised `LOGIN_RATE_LIMIT_MAX`, mirroring the existing rate-limit test's pattern, and its own `afterAll` cleaning up only its own fixtures — see Files to Change):
    - Create a fixture `Dealer` with `status: 'SUSPENDED'` and a linked `User` (role `DEALER`); `POST /auth/login` with correct credentials → `401`
    - Create a fixture `Dealer` with `status: 'ACTIVE'` and a linked `User` (role `DEALER`); `POST /auth/login` with correct credentials → `200` (regression check: the new suspension check doesn't block a normal dealer login)
    - Log in that same active-dealer fixture to obtain a refresh cookie, then update its `Dealer` row to `SUSPENDED` directly via Prisma, then `POST /auth/refresh` with the still-valid refresh cookie → `401` (proves the stale-session gap is closed)

**Frontend:**
- `AdminShell.test.jsx` — renders `<AdminShell />` inside a `MemoryRouter` with `api/dealers.js` mocked: at `/admin/dealers`, the list page's content renders; at `/admin/dealers/{id}`, the detail page's content renders (using a real route param via `MemoryRouter`, not a mocked `useParams`)
- `DealerListPage.test.jsx` — mocks `api/dealers.js`: renders a table row per dealer with name/region/status visible; filling and submitting the "New Dealer" form calls `createDealer` with the entered values
- `DealerDetailPage.test.jsx` — mocks `api/dealers.js`, rendered inside a `MemoryRouter` route with a real `:id` param (not a mocked `useParams`): renders the fetched dealer's fields; editing and saving the form calls `updateDealer`; clicking the status toggle calls `updateDealerStatus`

**CI:** both suites run via the existing `.github/workflows/ci.yml` jobs, no changes needed there.

## Database Schema

Added to `prisma/schema.prisma`:

```prisma
enum DealerStatus {
  ACTIVE
  SUSPENDED
}

model Dealer {
  id          String       @id @default(uuid())
  name        String
  region      String
  contactInfo String
  status      DealerStatus @default(ACTIVE)
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
  users       User[]

  @@map("dealers")
}
```

`User` model gains one field (the FK side of the relation):
```prisma
model User {
  id               String   @id @default(uuid())
  email            String   @unique
  passwordHash     String
  role             Role
  dealerId         String?
  dealer           Dealer?  @relation(fields: [dealerId], references: [id])
  refreshTokenHash String?
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@map("users")
}
```

Migration SQL (generated via `npx prisma migrate diff --from-schema-datamodel ./prisma/schema.prisma --to-schema-datamodel <draft-schema-with-the-above> --script`, run against the current committed schema — no live DB connection needed for this command; regenerated and confirmed byte-identical during plan verification):

```sql
-- CreateEnum
CREATE TYPE "DealerStatus" AS ENUM ('ACTIVE', 'SUSPENDED');

-- CreateTable
CREATE TABLE "dealers" (
    "id" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "region" TEXT NOT NULL,
    "contactInfo" TEXT NOT NULL,
    "status" "DealerStatus" NOT NULL DEFAULT 'ACTIVE',
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "dealers_pkey" PRIMARY KEY ("id")
);

-- AddForeignKey
ALTER TABLE "users" ADD CONSTRAINT "users_dealerId_fkey" FOREIGN KEY ("dealerId") REFERENCES "dealers"("id") ON DELETE SET NULL ON UPDATE CASCADE;
```

`ON DELETE SET NULL` matches `dealerId`'s existing nullable-optional semantics — deleting a `Dealer` row (not exposed by any endpoint in this task) would null out linked users' `dealerId` rather than fail or cascade-delete the login account.

## Documentation Updates

None needed. No API-spec or ERD doc exists yet in this repo (no `docs_root` configured, same as Tasks 01/02) — nothing else to update.

## Acceptance Criteria

1. `POST /dealers` (Admin) creates a `Dealer` and its linked `User` (role `DEALER`, correct `dealerId`) in one transaction, returns `201`.
2. `POST /dealers` with an email already in use returns `409`.
3. `GET /dealers` (Admin) returns the list of dealers, each with `region` and `status` visible.
4. `GET /dealers/:id` returns `404` for a non-existent id.
5. `PATCH /dealers/:id` (Admin) updates `name`/`region`/`contactInfo`.
6. `PATCH /dealers/:id/status` (Admin) toggles a dealer between `ACTIVE` and `SUSPENDED`.
7. `POST /auth/login` for a `DEALER` user linked to a `SUSPENDED` dealer returns `401` even with correct credentials.
8. `POST /auth/login` for a `DEALER` user linked to an `ACTIVE` dealer still succeeds (no regression from the new suspension check).
9. `POST /auth/refresh` also rejects with `401` when the caller's linked dealer has since become `SUSPENDED`, even with a still-valid, unexpired refresh cookie (closes the stale-session window).
10. All `/dealers` routes reject an unauthenticated caller with `401` and a non-Admin (Dealer-role) caller with `403`.
11. The frontend dealer list page renders dealers with region and status visible, submitting its "New Dealer" form calls `createDealer` and refreshes the list, and `AdminShell` correctly routes `/admin/dealers` to this page.
12. The frontend dealer detail page renders one dealer's fields, supports editing them, supports toggling status, and `AdminShell` correctly routes `/admin/dealers/:id` to this page.
13. `npm run lint` succeeds with zero errors in both `backend/` and `frontend/`.
14. GitHub Actions CI runs lint + test for both backend and frontend green on the PR, with the live Postgres service.

## Branch

`feature/fuel_petroleum-XXX-dealer-management` (created from `story/3-dealer-fuel-station-management-system`; ticket number filled in at Phase 4i)

## Change Log

| Date | Time | Person | Change |
|------|------|--------|--------|
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Plan created. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | a_sag_plan_verifier found 6 substantive issues, all fixed before user review: (1) `dealers.js`'s router guard was path-less (`router.use(authenticate, requireRole('ADMIN'))`), which would have turned any unmatched request into a false `401` once mounted — scoped it to `router.use('/dealers', ...)`. (2) `vite.config.js`'s dev proxy only covered `/auth`, so the new frontend pages couldn't reach the backend in `npm run dev` — added a `/dealers` proxy entry. (3) The planned `auth.test.js` additions didn't extend the existing `afterAll`, so new fixture rows would leak and break a second local test run — extended cleanup. (4) The two new login tests would have landed exactly on the CI rate limit with the existing block's 3 logins — moved them into their own isolated `describe` with a raised limit. (5) **Security gap:** the suspension check only covered `POST /auth/login`; `POST /auth/refresh` would keep minting access tokens for an already-suspended dealer for up to 7 days — added the same check to `/auth/refresh` and a new AC-9 covering it. (6) The `AdminShell` routing rewrite (the actual `/admin/dealers` → page wiring) had no test — added `AdminShell.test.jsx` and folded its coverage into AC-11/AC-12. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Re-verification found fix (3) was half-landed: it pointed at the existing `describe('auth routes', ...)`'s `afterAll`, but that hook is describe-scoped and runs before the new, later-declared suspended-dealer describe's fixtures even exist — extending it would be a no-op. Fixed: the new describe block now owns its own `afterAll`, cleaning up only its own fixture rows. Second re-verification pass confirms all 6 original fixes plus this correction. |

## Execution Tracking

- **Started:** 2026-09-15
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-dealer-management (created from story/3-dealer-fuel-station-management-system)
- **Collaborators:** (none yet)
