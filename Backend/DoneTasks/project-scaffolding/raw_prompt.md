# Task 01: Project Scaffolding

**Story:** Fuel_Petroleum#3 (Dealer & Fuel Station Management System)
**Story branch:** story/3-dealer-fuel-station-management-system

We're starting a brand-new web app: a Dealer & Fuel Station Management System with two roles (Admin, Dealer). Nothing exists in this repo yet, so this task lays the foundation everything else builds on.

Set up a backend service (Node.js + Express, PostgreSQL as the database, an ORM for schema/migrations) and a frontend SPA (React). Investigate current best-practice conventions for structuring an Express + Postgres API and a React app before deciding on folder layout — don't invent structure from scratch if there's an established pattern that fits.

What this needs to deliver:
- A runnable backend with a health-check endpoint that confirms it can reach the database
- A runnable frontend with a placeholder landing page
- Database connection and a migration tool wired up, with an empty baseline migration
- Linting and a test runner configured for both backend and frontend, with one passing example test each
- Basic CI (lint + test) so future PRs get checked automatically
- A root README explaining how to run both pieces locally

This is also where the app's visual identity gets set up, since every later task builds UI on top of it. The company is Ignite Petroleum Limited; its logo is at `E:/Fuel_Petroleum/ignite.jpeg` (flame-in-gear mark). Bring the logo into the frontend as a proper asset (export/crop it as needed — an SVG or a cleanly-cropped PNG for the header/login use, plus a favicon), and set up a shared theme (CSS custom properties or your framework's theming mechanism) with these tokens so nothing downstream hardcodes colors:

| Token | Hex | Use |
|---|---|---|
| `--color-primary` | `#DC3B2A` | Primary actions, active nav, links |
| `--color-accent` | `#F5A623` | Secondary accents, highlights, badges |
| `--color-background` | `#FDF9F5` | App background |
| `--color-surface` | `#FFFFFF` | Cards, panels, modals |
| `--color-text` | `#231A15` | Body text |

The placeholder landing page should use the logo and theme so there's something concrete to look at, not an unstyled page.

Keep this task to plumbing + branding only — no business logic, no auth, no data model yet. Every later task depends on this one being solid.

See full architecture plan: E:/Fuel_Petroleum/Planning_Tasks/Fuel_Petroleum-Plan-DealerFuelStationMgmt/spec.md
