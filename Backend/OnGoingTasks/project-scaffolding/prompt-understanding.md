# Task 01: Project Scaffolding — Understanding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system
**Change Class:** FEATURE (greenfield — no existing behavior to preserve)

## What This Delivers

The foundation for a two-role (Admin/Dealer) fuel station management web app, for Ignite Petroleum Limited. Nothing exists in the repo yet beyond a README and config files, so this is pure scaffolding: no business logic, no auth, no data model.

**Backend:** Node.js + Express API, PostgreSQL database, Prisma as the ORM/migration tool. A `/health` endpoint that confirms DB connectivity.

**Frontend:** React SPA (Vite-based) with a placeholder landing page that actually uses the Ignite Petroleum branding, not an unstyled page.

**Branding (from the spec):** logo at `E:/Fuel_Petroleum/ignite.jpeg`, brought into the frontend as a proper asset (cropped/exported version for header/login use, plus a favicon). Theme tokens as CSS custom properties:

| Token | Hex | Use |
|---|---|---|
| `--color-primary` | `#DC3B2A` | Primary actions, active nav, links |
| `--color-accent` | `#F5A623` | Secondary accents, highlights, badges |
| `--color-background` | `#FDF9F5` | App background |
| `--color-surface` | `#FFFFFF` | Cards, panels, modals |
| `--color-text` | `#231A15` | Body text |

**Tooling:** ESLint (+ Prettier) and a test runner (Jest or Vitest, matching whatever fits Express/React best) for both backend and frontend, each with one passing example test. Basic GitHub Actions CI running lint + test on PRs. A root README covering local setup for both pieces.

## Out of Scope

No auth, no dealer/station/product data model, no business logic. Those are Tasks 02+ in the same story.

## Applicable Rules

No project-specific coding standards are installed yet (`standards_location` is unset in `.claude/config_hints.json` — this is a brand-new repo). Follow standard Express/React/Prisma community conventions until the project's own rules exist.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
