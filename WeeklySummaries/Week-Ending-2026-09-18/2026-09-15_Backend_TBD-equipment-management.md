# Equipment Management

**Date:** 2026-09-15 | **Developer:** noumanmuzaffar007@gmail.com | **Platform:** Backend

**What:** Dealers had no way to record the physical tanks, dispensers, and nozzles at their stations, and there was no equipment-level model at all for later Sales/Reporting tasks to reference.

**Fix:** Added a polymorphic `Equipment` model (TANK/DISPENSER/NOZZLE) with dealer-scoped CRUD (reusing the `dealerScope` middleware and relation-filter list-scoping patterns from Tasks 02-04), a self-relation for nozzle-to-dispenser hierarchy with explicit `onDelete: Restrict`, and a read-only cross-network Admin view. Frontend adds per-station equipment management (Dealer) and a filterable read-only equipment list (Admin). PR: https://github.com/Nouman810/Fuel_Petroleum/pull/8, CI green (backend + frontend jobs).
