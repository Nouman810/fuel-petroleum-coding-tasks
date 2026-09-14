# Execution Plan: Task 01 — Project Scaffolding

**Branch:** `feature/fuel_petroleum-XXX-project-scaffolding` (branched from `story/3-dealer-fuel-station-management-system`, ticket TBD — filled in at Phase 4i)
**PR target:** `story/3-dealer-fuel-station-management-system` (not `main`)

## Summary

Stand up the two-service skeleton (Express/Prisma/PostgreSQL backend, React/Vite frontend) plus shared tooling and branding, with zero business logic. Every later task in the story builds on this.

## Approach & Trade-offs

- **Monorepo, two top-level folders** (`backend/`, `frontend/`) in the single `Fuel_Petroleum` repo — matches the modular-monolith decision in the spec; no need for separate repos or a workspace tool (Turborepo/Nx) at this size.
- **Prisma over raw `pg` or another ORM** — per spec.md's architecture section; gives migrations + typed client for free, which every later CRUD-heavy task (dealers, stations, orders, etc.) will lean on.
- **Vite + React over Next.js** — the spec calls for a plain SPA behind an Express API, not SSR; Vite is the lighter, faster-to-scaffold choice for that shape.
- **Jest for backend, Vitest for frontend** — Jest is the conventional pairing with Express/Node; Vitest is Vite's native test runner and avoids a second bundler config for the frontend.
- **Logo used as-is (no image-processing step)** — `ignite.jpeg` already sits on a cream background matching `--color-background`, so it drops into the header/login and favicon slots without cropping. Introducing a design tool or image library (`sharp`, etc.) for a one-time asset copy would be over-engineering for this task; revisit if a true multi-resolution icon set is needed later.
- **Docker Compose for local Postgres, a real Postgres service container in CI** — the health-check endpoint's job is to prove real DB connectivity, so both local dev and CI need an actual database, not a mock.

## Files to Change

**Root**
- `README.md` — rewrite with setup/run instructions for both services
- `.gitignore` — add `node_modules/`, `dist/`, `.env`, `coverage/`
- `docker-compose.yml` — new, one `postgres` service for local dev
- `.github/workflows/ci.yml` — new, two jobs (backend, frontend), each: checkout → setup-node → npm ci → lint → test; backend job gets a `postgres` service container

**Backend** (`backend/`)
- `package.json` — express, @prisma/client, cors, dotenv; devDeps: prisma, jest, supertest, eslint
- `.env.example` — `DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fuel_petroleum`
- `eslint.config.js` — flat config, Node environment
- `prisma/schema.prisma` — datasource (`postgresql`, env `DATABASE_URL`) + generator (`prisma-client-js`); no models yet
- `prisma/migrations/` — generated baseline migration (empty, since no models)
- `src/prismaClient.js` — exports a singleton `PrismaClient` instance
- `src/app.js` — Express app: JSON middleware, CORS, mounts `/health` router
- `src/server.js` — reads `PORT` from env, starts the app from `app.js`
- `src/routes/health.js` — `GET /health` → runs `prisma.$queryRaw\`SELECT 1\`` , returns `{ status: "ok", db: "connected" }` (200) or `{ status: "error", db: "disconnected" }` (503) on failure
- `src/routes/health.test.js` — supertest hitting `/health`, asserting 200 + shape (test env points at the same Postgres via `DATABASE_URL`, or CI's service container)
- `jest.config.js` — node test environment

**Frontend** (`frontend/`)
- `package.json` — react, react-dom; devDeps: vite, @vitejs/plugin-react, vitest, @testing-library/react, @testing-library/jest-dom, jsdom, eslint
- `vite.config.js` — React plugin + Vitest config (`environment: 'jsdom'`)
- `index.html` — root HTML, favicon link to the logo asset
- `eslint.config.js` — flat config, React + browser environment
- `public/ignite-logo.jpeg` — copy of `E:/Fuel_Petroleum/ignite.jpeg` (also used as favicon source)
- `src/styles/theme.css` — `:root { --color-primary: #DC3B2A; --color-accent: #F5A623; --color-background: #FDF9F5; --color-surface: #FFFFFF; --color-text: #231A15; }` plus a couple of base element rules (`body`, `a`) using the tokens
- `src/main.jsx` — React root, imports `theme.css`
- `src/App.jsx` — placeholder landing page: logo image, "Ignite Petroleum Limited" heading, "Dealer & Fuel Station Management System" subheading, styled via the theme tokens (surface card on background)
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

Running `npx prisma migrate dev --name init` against this schema produces an empty baseline migration (no `CREATE TABLE` statements) — this is the "empty baseline migration" the raw prompt asks for. Task 02+ add models incrementally on top of this baseline.

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

## Execution Tracking

- **Started:** 2026-09-14
- **Developer:** noumanmuzaffar007@gmail.com
- **Branch:** feature/fuel_petroleum-XXX-project-scaffolding
- **Collaborators:** (none yet)
