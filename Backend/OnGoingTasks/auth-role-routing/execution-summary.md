## Pull Request
- *PR not yet created*

## Current State
- **Phase:** 3 (Code)
- **Branch:** feature/fuel_petroleum-XXX-auth-role-routing
- **Worktree:** false
- **Local Branch:** feature/fuel_petroleum-XXX-auth-role-routing
- **Remote Branch:** feature/fuel_petroleum-XXX-auth-role-routing
- **Reconciliation Result:** created in-place branch (from feature/fuel_petroleum-XXX-project-scaffolding, not main — Task 01's PR #4 is unmerged and this task needs its code)
- **Last Action:** Resumed 2026-09-15. Backend core files already written (password.js, tokens.js, middleware/auth.js, routes/auth.js, seed.js, migration, schema, app.js wiring) but no test files yet. Frontend untouched beyond scaffolding. Backend lint green. Resumed without DB smoke test (user override) — no Postgres/Docker available on this machine, no backend/.env yet; DB-dependent tests (auth.test.js, seed.test.js, middleware/auth.test.js — none of which exist yet) deferred until a database is available.

## Q&A Log
- Q: Refresh token storage/rotation strategy? → A: httpOnly secure cookie, DB-hashed (jti), rotated on every use. Access token in-memory only on the client.
- Q: How to handle User.dealerId with no Dealer table yet? → A: Plain nullable UUID column, no FK. Task 03 adds the Dealer table; a later migration adds the FK.
- Q: Ticket-first or ticket-late? → A: Ticket-late (same as Task 01) — ticket number obtained at Phase 4i.
- Plan verifier (a_sag_plan_verifier) found 4 issues, all fixed before approval: missing Jest globals in backend eslint.config.js, no automated AC-7 verification, nested-router test conflict, migration folder naming convention.

## Next Steps
- Implement backend: deps, schema+migration, utils (password/tokens), middleware (auth.js), routes (auth.js), seed.js, tests.
- Implement frontend: react-router-dom, AuthContext, api client, LoginPage/AdminShell/DealerShell/RequireRole, tests.
- Run tests, flip acceptance_criteria.json flags as each is verified.
