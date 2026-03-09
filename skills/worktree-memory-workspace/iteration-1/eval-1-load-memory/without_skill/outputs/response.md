Memory loaded from `../memory.md`. Here's where things stand:

**Phase 2: API implementation** — in progress

**Stack:** Go + Chi router, SQLite storage

**Key patterns:**
- Handler tests use `httptest.NewRecorder`
- All DB access through `Store` interface
- Chi middleware order: auth before logging

**Recent activity:**
- 2026-03-01: Project init, phase 1 complete
- 2026-03-05: Phase 2 started, API routes scaffolded

You're in the `fix-something` worktree. What are we fixing?
