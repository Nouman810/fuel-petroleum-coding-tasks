## Pull Request
- *PR not yet created*

## Current State
- **Phase:** 1 (Prompt Understanding) → awaiting user approval
- **Branch:** not yet created (will branch from feature/fuel_petroleum-XXX-project-scaffolding)
- **Last Action:** Wrote prompt-understanding.md after reading spec.md, backend schema.prisma/app.js/package.json, and frontend main.jsx.

## Q&A Log
- Q: Refresh token storage/rotation strategy? → A: httpOnly secure cookie, DB-hashed, rotated on every use. Access token in-memory only on the client.
- Q: How to handle User.dealerId with no Dealer table yet? → A: Plain nullable UUID column, no FK. Task 03 adds the Dealer table; a later migration adds the FK.
- Q: Ticket-first or ticket-late? → A: Ticket-late (same as Task 01) — ticket number obtained at Phase 4i.

## Next Steps
- Get user approval on prompt-understanding.md, then proceed to Phase 2 (execution_plan.md), including confirming the branch-from-scaffolding-branch strategy in the plan.
