# Execution Plan: Task 02 — Auth & Role-Based Routing

**Branch:** `feature/fuel_petroleum-XXX-auth-role-routing` (branched from `feature/fuel_petroleum-XXX-project-scaffolding`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system`
**Change Class:** FEATURE

## Summary

Build the shared login flow and the two infrastructure pieces every later module depends on: a role guard and a dealer-scoping middleware factory. Backend issues a JWT access token + rotated httpOnly refresh-token cookie; frontend gains routing, a login page, and role-gated Admin/Dealer shells.

## Approach & Trade-offs

- **Refresh token = signed JWT carrying `{sub, jti}`, DB stores only the current `jti`'s hash** (not a plain opaque random token). Verifying the JWT signature+expiry needs no DB hit; the stored `jti` hash is compared only to detect rotation/reuse (an old, already-exchanged refresh token's `jti` won't match what's stored, so it's rejected even though its signature is still valid until expiry). One active session per user — acceptable for this scaffolding-stage task; multi-device sessions aren't in the spec.
- **`bcryptjs` instead of `bcrypt`.** Same hashing algorithm; the native `bcrypt` package needs a C++ build toolchain, which caused real friction on this Windows dev machine during Task 01 (Prisma's own native engine already required workarounds). `bcryptjs` is pure JS, same security properties, no native compile step.
- **Vite dev-server proxy (`/auth` → `http://localhost:3000`) instead of relying on cross-origin CORS+cookies for local dev.** A cross-site `Set-Cookie` for the refresh token would need `SameSite=None; Secure`, which requires HTTPS — not available in local dev. Proxying makes the browser see same-origin requests, so the cookie sets normally under the default `SameSite=Lax`. Backend CORS is still configured properly (`credentials: true`, explicit `FRONTEND_URL` origin) for any non-proxied deployment later; the proxy is a dev-convenience layer on top, not a replacement.
- **Access token held in memory only (React state/context), never in localStorage.** Reduces the XSS blast radius given the refresh token is already httpOnly. Trade-off: a full-page reload loses the in-memory token, so the app does a silent `/auth/refresh` on mount (cookie sent automatically) to restore the session without forcing re-login — the access token's JWT payload is decoded client-side (no signature check needed, it just came from our own `/refresh` response) to repopulate `{id, email, role, dealerId}`.
- **`User.dealerId` is a plain nullable `String` column, no FK** — confirmed with user in Phase 1. Task 03 introduces the `Dealer` table; a later migration adds the constraint.
- **Dealer-scoping middleware is a factory, not tied to any resource** (`dealerScope(getOwnerDealerId)`), since no station/order/complaint table exists yet. Verified in this task only via a unit test with a mock ownership-lookup function; real wiring happens per-resource starting Task 03/04.

## Files to Change

**Backend** (`backend/`)
- `package.json` — add deps: `jsonwebtoken`, `bcryptjs`, `express-rate-limit`, `zod`, `cookie-parser`; add `"prisma": { "seed": "node prisma/seed.js" }` block
- `.env.example` — add `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `ACCESS_TOKEN_TTL=15m`, `REFRESH_TOKEN_TTL=7d`, `FRONTEND_URL=http://localhost:5173`, `COOKIE_SECURE=false`, `BCRYPT_SALT_ROUNDS=10`, `LOGIN_RATE_LIMIT_MAX=5`, `LOGIN_RATE_LIMIT_WINDOW_MS=900000`
- `jest.config.js` — add `setupFiles: ['<rootDir>/jest.setup.js']`
- `jest.setup.js` — new, `require('dotenv').config()` so local `jest` runs pick up `.env` the same way `server.js` does (tests `require('../app')` directly, bypassing `server.js`)
- `prisma/schema.prisma` — add `Role` enum (`ADMIN`, `DEALER`) and `User` model (see Database Schema below)
- `prisma/migrations/20260915000000_add_user_auth/migration.sql` — new, generated via `npx prisma migrate diff --from-empty --to-schema-datamodel <draft> --script` against the target schema (no live DB needed for this command); see Database Schema section for the exact output. (14-digit `YYYYMMDDHHMMSS` prefix, matching the existing `20260914120000_init` convention.)
- `prisma/seed.js` — new, exports `seedUsers(prisma)` (upserts one `ADMIN` and one `DEALER` user, `dealerId: null`) and self-invokes it when run directly (`if (require.main === module)`) so `npx prisma db seed` still works
- `prisma/seed.test.js` — new, calls `seedUsers` against the live test DB, asserts exactly one `ADMIN` and one `DEALER` row exist for the seeded emails, cleans up in `afterAll`. Satisfies AC-7 through the normal `npm test` run — no separate CI step needed.
- `eslint.config.js` — extend the `**/*.test.js` globals block with `beforeAll`, `afterAll`, `beforeEach`, `afterEach`, `jest` (currently only declares `describe`/`it`/`expect`, so the new test files' Jest lifecycle hooks would fail `no-undef`)
- `src/utils/password.js` — new, `hashPassword(plain)` / `comparePassword(plain, hash)` via `bcryptjs`
- `src/utils/tokens.js` — new, `signAccessToken(user)`, `signRefreshToken(user)` (returns `{ token, jti }`), `hashJti(jti)` (sha256), `verifyAccessToken(token)`, `verifyRefreshToken(token)`
- `src/middleware/auth.js` — new, `authenticate` (verifies `Authorization: Bearer`, sets `req.user`), `requireRole(...roles)`, `dealerScope(getOwnerDealerId)` (no-ops for non-Dealer roles; 403 on mismatch)
- `src/routes/auth.js` — new, `POST /auth/login` (zod-validated, rate-limited, bcrypt-verified), `POST /auth/refresh` (rotates), `POST /auth/logout` (clears cookie + DB hash)
- `src/routes/auth.test.js` — new, see Test Plan
- `src/middleware/auth.test.js` — new, unit tests for `requireRole`/`dealerScope` with mock req/res/next
- `src/app.js` — wire `cookie-parser`, tighten `cors()` to `{ origin: process.env.FRONTEND_URL, credentials: true }`, mount `authRouter`

**Root**
- `.github/workflows/ci.yml` — backend job `env:` block gains `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `FRONTEND_URL`, `BCRYPT_SALT_ROUNDS`, `LOGIN_RATE_LIMIT_MAX`, `LOGIN_RATE_LIMIT_WINDOW_MS`, `COOKIE_SECURE=false` (dummy values, this is CI not production)
- `README.md` — add a short "Seeding" note (`npx prisma db seed`) under backend setup

**Frontend** (`frontend/`)
- `package.json` — add `react-router-dom`
- `vite.config.js` — add `server.proxy: { '/auth': 'http://localhost:3000' }`
- `src/api/client.js` — new, `apiFetch`, module-scoped access-token holder, 401-triggers-one-retry-after-refresh logic
- `src/api/jwt.js` — new, `decodeJwtPayload(token)` (base64 decode, no verification — trusted because it came from our own response)
- `src/context/AuthContext.jsx` — new, `AuthProvider` (on-mount silent refresh), `useAuth()` hook exposing `{ user, login, logout, initializing }`
- `src/routes/RequireRole.jsx` — new, redirects to `/` if unauthenticated or wrong role
- `src/routes/RequireRole.test.jsx` — new, wraps in `MemoryRouter initialEntries={[...]}` with a mocked `useAuth`: an Admin `user` renders the protected child at `/admin`, a Dealer `user` hitting `/admin` is redirected to `/` (covers AC-9's both-roles path — see note on router placement below)
- `src/routes/LoginPage.jsx` — new, email/password form, reuses the branded card layout from the current `App.jsx`, redirects to `/admin` or `/dealer` on success based on returned role
- `src/routes/AdminShell.jsx` — new, placeholder ("Admin Dashboard" heading, themed)
- `src/routes/DealerShell.jsx` — new, placeholder ("Dealer Dashboard" heading, themed)
- `src/main.jsx` — `BrowserRouter` moves here (wraps `<App />`), so `App` itself stays router-agnostic and testable with `MemoryRouter`
- `src/App.jsx` — rewritten: `AuthProvider` + `Routes` only, no `BrowserRouter` (that lives in `main.jsx` — see above) (`/` → `LoginPage`, `/admin/*` → `RequireRole role="ADMIN"`, `/dealer/*` → `RequireRole role="DEALER"`). The old placeholder-landing-page content is superseded by `LoginPage` (branding carries over: same logo, heading, card style).
- `src/App.test.jsx` — rewritten (FEATURE change, `App`'s own contract changed from static landing page to router root): renders `<App />` wrapped in `MemoryRouter` (no nested-router conflict now that `BrowserRouter` isn't inside `App`), asserts the login form renders at `/`
- `src/routes/LoginPage.test.jsx` — new, form renders, submit calls the auth context's `login`
- `src/context/AuthContext.test.jsx` — new, mocks `fetch` for `/auth/refresh` to prove on-mount session restore

**Note (all new frontend test files):** import `describe`/`it`/`expect`/`vi`/`beforeEach` explicitly from `'vitest'` (as `App.test.jsx` already does) rather than relying on globals — `frontend/eslint.config.js` doesn't declare vitest globals even though `vite.config.js` sets `test.globals: true`.

## Test Plan

**Backend:**
- `auth.test.js` — creates its own fixture Admin/Dealer users in `beforeAll` (unique emails, not the seed script's accounts) against the live Postgres, cleans up in `afterAll`:
  - Valid Admin login → 200, body has `accessToken` + `user.role === 'ADMIN'`, response sets a `refreshToken` cookie
  - Wrong password → 401
  - Valid refresh cookie → `POST /auth/refresh` returns 200 with a new `accessToken`, and the DB's stored `jti` hash changed (rotation)
  - Rate limit: with `LOGIN_RATE_LIMIT_MAX` set low via `jest.resetModules()` + env override in this file, the `(max+1)`th login attempt within the window returns 429
- `middleware/auth.test.js` — unit tests, no DB/HTTP needed:
  - `requireRole('ADMIN')` calls `next()` for an Admin `req.user`, responds 403 for a Dealer
  - `dealerScope(mockGetOwnerDealerId)` calls `next()` unconditionally for an Admin `req.user`; for a Dealer, calls `next()` when the mock's returned `dealerId` matches `req.user.dealerId`, responds 403 when it doesn't
- `prisma/seed.test.js` — calls `seedUsers(prisma)` against the live DB, queries for the seeded emails, asserts exactly one `ADMIN` row and one `DEALER` row exist, deletes them in `afterAll`

**Frontend:**
- `App.test.jsx` — renders `<App />` wrapped in `MemoryRouter`, asserts the login form (email input, password input, submit button) is present at `/`
- `LoginPage.test.jsx` — fills the form, submits, asserts the mocked `login` function was called with the entered credentials
- `RequireRole.test.jsx` — with a mocked `useAuth`: an Admin `user` renders the protected child under `/admin/*`; a Dealer `user` hitting `/admin/*` is redirected to `/` — covers AC-9
- `AuthContext.test.jsx` — mocks `fetch` to resolve `/auth/refresh` with a fake access token, asserts `useAuth().user` is populated after mount without any explicit login call

**CI:** both suites run via the existing `.github/workflows/ci.yml` jobs; backend gets the new JWT/rate-limit/cookie env vars added to its `env:` block.

## Database Schema

Added to `prisma/schema.prisma`:

```prisma
enum Role {
  ADMIN
  DEALER
}

model User {
  id               String   @id @default(uuid())
  email            String   @unique
  passwordHash     String
  role             Role
  dealerId         String?
  refreshTokenHash String?
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@map("users")
}
```

Migration SQL (generated via `npx prisma migrate diff --from-empty --to-schema-datamodel <draft-schema> --script`, run against the schema above — no live DB connection needed for this command, so this is real tool output, not hand-typed):

```sql
-- CreateSchema
CREATE SCHEMA IF NOT EXISTS "public";

-- CreateEnum
CREATE TYPE "Role" AS ENUM ('ADMIN', 'DEALER');

-- CreateTable
CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "passwordHash" TEXT NOT NULL,
    "role" "Role" NOT NULL,
    "dealerId" TEXT,
    "refreshTokenHash" TEXT,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");
```

`prisma/seed.js` exports `seedUsers(prisma)`, which upserts (by email) one `ADMIN` and one `DEALER` user with `dealerId: null` — idempotent, so `npx prisma db seed` is safe to re-run. `prisma/seed.test.js` calls this same function directly (see Test Plan), which is how AC-7 gets verified in the normal `npm test` run instead of a separate CI step.

## Documentation Updates

- `README.md`: add a one-line "Seeding" step to the backend setup section (`npx prisma db seed` after migrating).
- No API-spec or ERD doc exists yet in this repo (no `docs_root` configured) — nothing else to update.

## Acceptance Criteria

1. `POST /auth/login` with valid Admin credentials returns 200 with an `accessToken` and `user.role === "ADMIN"`, and sets a `refreshToken` cookie.
2. `POST /auth/login` with an invalid password returns 401.
3. `POST /auth/refresh` with a valid refresh cookie returns 200 with a new `accessToken`, and rotates the stored token hash.
4. `requireRole('ADMIN')` middleware rejects a Dealer-role request with 403 (unit test).
5. `dealerScope(...)` middleware lets Admin requests through unconditionally and rejects a Dealer request with 403 when the resource's owning dealer doesn't match the requester (unit test).
6. `POST /auth/login` is rate-limited: exceeding the configured max attempts within the window returns 429.
7. `prisma/seed.js` creates exactly one `ADMIN` user and one `DEALER` user when run against a live database.
8. Frontend renders a login form (email, password, submit) at `/`.
9. After a successful login, an Admin-role user reaches `/admin/*`; a Dealer-role user attempting `/admin/*` is redirected away.
10. On mount, if a valid refresh cookie exists, the app restores the authenticated session without requiring the user to log in again.
11. `npm run lint` succeeds with zero errors in both `backend/` and `frontend/`.
12. GitHub Actions CI runs lint + test for both backend and frontend green on the PR, with the live Postgres service.

## Branch

`feature/fuel_petroleum-XXX-auth-role-routing` (created from `feature/fuel_petroleum-XXX-project-scaffolding`; ticket number filled in at Phase 4i)

## Change Log

| Date | Time | Person | Change |
|------|------|--------|--------|
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | Plan created. |
| 2026-09-15 | - | noumanmuzaffar007@gmail.com | a_sag_plan_verifier found 4 issues, fixed before user review: (1) backend `eslint.config.js` didn't declare Jest lifecycle globals (`beforeAll`/`afterAll`/`jest`) for `**/*.test.js`, would've failed lint — added to Files to Change. (2) AC-7 (seed creates 1 Admin + 1 Dealer) had no automated verification path — refactored `seed.js` to export `seedUsers(prisma)` and added `prisma/seed.test.js` calling it directly, instead of a separate CI step. (3) Plan had `BrowserRouter` inside `App.jsx` while the Test Plan said to wrap `App` in `MemoryRouter` for tests — nested-router conflict. Moved `BrowserRouter` to `main.jsx`, kept `App` router-agnostic. Also added a missing `RequireRole.test.jsx` to cover AC-9 (was otherwise untested). (4) Migration folder name `20260915_add_user_auth` didn't match the existing 14-digit `YYYYMMDDHHMMSS` convention — renamed to `20260915000000_add_user_auth`. |

## Execution Tracking

- **Started:** 2026-09-15
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-auth-role-routing
- **Collaborators:** (none yet)
