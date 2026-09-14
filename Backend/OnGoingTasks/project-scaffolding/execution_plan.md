# Execution Plan: Task 01 — Project Scaffolding

**Branch:** `feature/fuel_petroleum-XXX-project-scaffolding` (branched from `story/3-dealer-fuel-station-management-system`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system` (not `main`)
**Change Class:** FEATURE

## Summary

Stand up the two-service skeleton (Express/Prisma/PostgreSQL backend, React/Vite frontend) plus shared tooling and branding, with zero business logic. Every later task in the story builds on this.

## Approach & Trade-offs

- **Monorepo, two top-level folders** (`backend/`, `frontend/`) in the single `Fuel_Petroleum` repo — matches the modular-monolith decision in the spec; no need for separate repos or a workspace tool (Turborepo/Nx) at this size.
- **Prisma over raw `pg` or another ORM** — per spec.md's architecture section; gives migrations + typed client for free, which every later CRUD-heavy task (dealers, stations, orders, etc.) will lean on.
- **Vite + React over Next.js** — the spec calls for a plain SPA behind an Express API, not SSR; Vite is the lighter, faster-to-scaffold choice for that shape.
- **Jest for backend, Vitest for frontend** — Jest is the conventional pairing with Express/Node; Vitest is Vite's native test runner and avoids a second bundler config for the frontend.
- **Logo is cropped once via a local script, not shipped as a runtime dependency.** The source JPEG is a 1024x1024 frame with large padding and a drop shadow — used as-is it would show as a visible off-white box on the white `--color-surface` card, and the icon alone is illegible at favicon size. Fix: a one-time PowerShell script (`System.Drawing`, already on the machine, no new package) trims the JPEG to (a) a tight `logo-full.png` (icon + wordmark, 696x464) for the header/login, and (b) a square icon crop centered on the theme's `--color-background` (`#FDF9F5`) canvas, resized to 180/32px, for the favicon and touch-icon. This was verified visually before finalizing the plan — the 32px crop is legible. The output PNGs are committed as static assets; no image-processing library becomes a project dependency.
- **Docker Compose for local Postgres, a real Postgres service container in CI** — the health-check endpoint's job is to prove real DB connectivity, so both local dev and CI need an actual database, not a mock.

## Files to Change

**Root**
- `README.md` — rewrite with setup/run instructions for both services (no machine-local paths — the logo source path is referenced only in this plan/task doc, never in README)
- `.gitignore` — add `node_modules/`, `dist/`, `.env`, `coverage/`
- `docker-compose.yml` — new, one `postgres:16` service for local dev
- `.github/workflows/ci.yml` — new, two jobs (backend, frontend), each: checkout → setup-node → npm ci → lint → test; backend job gets a `postgres:16` service container and runs `npx prisma generate` before tests (schema must be generated before `PrismaClient` can be imported)

**Backend** (`backend/`)
- `package.json` — express, @prisma/client, cors, dotenv; devDeps: prisma, jest, supertest, eslint, prettier
- `.env.example` — `DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fuel_petroleum`, `PORT=3000`
- `eslint.config.js` — flat config, Node environment
- `.prettierrc` — shared formatting config
- `prisma/schema.prisma` — datasource (`postgresql`, env `DATABASE_URL`) + generator (`prisma-client-js`); no models yet
- `prisma/migrations/` — baseline migration created with `prisma migrate dev --create-only --name init` (see Database Schema section — a model-less schema has no diff, so `--create-only` is required to force an empty `migration.sql` to exist; plain `migrate dev` would report "already in sync" and create nothing)
- `src/prismaClient.js` — exports a singleton `PrismaClient` instance
- `src/app.js` — Express app: JSON middleware, CORS, mounts `/health` router
- `src/server.js` — reads `PORT` from env, starts the app from `app.js`
- `src/routes/health.js` — `GET /health` → runs `prisma.$queryRaw\`SELECT 1\`` , returns `{ status: "ok", db: "connected" }` (200) or `{ status: "error", db: "disconnected" }` (503) on failure
- `src/routes/health.test.js` — supertest hitting `/health`, asserting 200 + shape (test env points at the same Postgres via `DATABASE_URL`, or CI's service container)
- `jest.config.js` — node test environment

**Frontend** (`frontend/`)
- `package.json` — react, react-dom; devDeps: vite, @vitejs/plugin-react, vitest, @testing-library/react, @testing-library/jest-dom, jsdom, eslint, eslint-plugin-react, eslint-plugin-react-hooks, globals, prettier
- `vite.config.js` — React plugin + Vitest config (`environment: 'jsdom'`, `setupFiles: ['./src/setupTests.js']`)
- `src/setupTests.js` — `import '@testing-library/jest-dom'` (registers the DOM matchers the App test uses)
- `index.html` — root HTML, favicon links (`icon-32.png`, `icon-180.png` apple-touch-icon)
- `eslint.config.js` — flat config, React + browser environment (via `eslint-plugin-react`/`eslint-plugin-react-hooks`/`globals`)
- `.prettierrc` — shared formatting config (same as backend)
- `public/logo-full.png` — tight crop of the icon+wordmark lockup (696x464), cropped from `ignite.jpeg` via the one-time script described above; used in the header/login
- `public/icon-32.png`, `public/icon-180.png` — square favicon/touch-icon, cropped and padded from the same source (see Approach)
- `src/styles/theme.css` — `:root { --color-primary: #DC3B2A; --color-accent: #F5A623; --color-background: #FDF9F5; --color-surface: #FFFFFF; --color-text: #231A15; }` plus a couple of base element rules (`body`, `a`) using the tokens
- `src/main.jsx` — React root, imports `theme.css`
- `src/App.jsx` — placeholder landing page: `logo-full.png`, "Ignite Petroleum Limited" heading, "Dealer & Fuel Station Management System" subheading, styled via the theme tokens (surface card on background)
- `src/App.test.jsx` — renders `<App />`, asserts the heading text is present

## Test Plan

- **Backend:** `backend/src/routes/health.test.js` — `GET /health` returns 200 and `{ status: "ok" }` when the DB is reachable. This is the one required passing example test; it doubles as real verification that Prisma can reach Postgres.
- **Frontend:** `frontend/src/App.test.jsx` — renders without throwing and shows the expected heading text. Confirms the placeholder page and theme wiring don't crash.
- **CI:** both suites run on every PR via `.github/workflows/ci.yml`; backend test run gets a live `postgres:16` service container so `/health` is tested against a real database, not mocked.

## Database Schema

No models yet (by design — this task is plumbing only). `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

A model-less schema has no diff to apply, so plain `prisma migrate dev` reports "already in sync" and creates no migration at all. To get the "empty baseline migration" the raw prompt asks for, use `npx prisma migrate dev --create-only --name init` (forces an empty `migration.sql` to be written), then `npx prisma migrate deploy` to apply it. Task 02+ add models incrementally on top of this baseline.

## Documentation Updates

- `README.md` (this repo's root): full rewrite — prerequisites (Node, Docker for local Postgres), backend setup (`cp .env.example .env`, `npm install`, `npx prisma migrate dev`, `npm run dev`), frontend setup (`npm install`, `npm run dev`), how to run tests for each.
- No API-spec or ERD doc exists yet in this greenfield repo (no `docs_root` configured) — nothing else to update.

## Acceptance Criteria

1. `GET /health` returns 200 with a body indicating the database is connected, when run against a live Postgres instance.
2. Backend has one passing Jest test (the health-check test).
3. Frontend renders a landing page showing the Ignite Petroleum logo, the app name, and uses the five theme tokens (verifiable via computed styles or by inspecting `theme.css` is imported and applied).
4. Frontend has one passing Vitest test.
5. `npm run lint` succeeds with zero errors in both `backend/` and `frontend/`.
6. GitHub Actions CI workflow runs lint + test for both backend and frontend on a PR, including a live Postgres service for the backend job.
7. Root `README.md` documents how to set up and run both services locally, including the database.

## Branch

`feature/fuel_petroleum-XXX-project-scaffolding` (created from `story/3-dealer-fuel-station-management-system`; ticket number filled in at Phase 4i)

## Change Log

| Date | Time | Person | Change |
|------|------|--------|--------|
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | `npm create vite@latest` now scaffolds with `oxlint` by default instead of ESLint. Swapped it out for the planned ESLint flat config (+ `eslint-plugin-react`, `eslint-plugin-react-hooks`, `globals`) to keep both services on one linting story, per the plan. |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | No Docker/Postgres/package-manager available in the dev shell. Health-check test and Prisma migration are written correctly per plan but verified via CI's live postgres service container rather than locally in this session (user-confirmed). |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | `npm install prisma`/`@prisma/client` with no version pin resolved to `8.0.0-rc.15` — a pre-release "Prisma Developer Platform" CLI with a completely different command tree (no `generate`/`migrate` commands) and, on the last-stable `7.x` line, a breaking change removing `datasource.url` from `schema.prisma` in favor of a `prisma.config.ts` + driver-adapter pattern. Pinned both packages to `6.19.3` (last stable release matching the plan's simple `url = env("DATABASE_URL")` schema, no adapter needed) to keep this task's scope to plumbing only. Migrating to Prisma 7's driver-adapter pattern is out of scope here. |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | This machine's Kaspersky Endpoint Security TLS-inspection proxy caused every npm install to fail instantly with `SELF_SIGNED_CERT_IN_CHAIN` (masked as a hang because output was piped through `tail`, losing npm's real exit code). Fixed by exporting the Kaspersky root CA from the Windows cert store and setting `NODE_EXTRA_CA_CERTS` for npm/node commands — not a code change, but worth recording since it'll hit every future task on this machine until set permanently. |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | Could not run `prisma migrate dev --create-only` (needs a live DB connection even to create an empty migration). Hand-wrote `prisma/migrations/migration_lock.toml` and `prisma/migrations/20260914120000_init/migration.sql` (empty) matching Prisma's exact on-disk convention instead. Functionally identical to what the CLI would produce for a model-less schema. |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | `npm create vite@latest` resolved a fresh `eslint@10.10.0`, which conflicts with `eslint-plugin-react@7.37.5`'s peer range (`^3‖...‖^9.7`, no 10 support yet). Pinned frontend's `eslint`/`@eslint/js` to `^9` to satisfy the peer dependency rather than forcing past the conflict. |
| 2026-09-14 | - | noumanmuzaffar007@gmail.com | `@testing-library/jest-dom` v7's `setupTests.js` import calls `expect.extend(...)` at module load, which threw `ReferenceError: expect is not defined` because Vitest doesn't inject `describe`/`it`/`expect` as globals by default. Added `test.globals: true` to `vite.config.js` (standard Jest-compatible Vitest config) to fix it. |

## Execution Tracking

- **Started:** 2026-09-14
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-project-scaffolding
- **Collaborators:** (none yet)
