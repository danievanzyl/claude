Memory loaded for **testproject**.

You're in **Phase 2: API implementation** (in progress).

Key context:
- Stack: Go + Chi router, SQLite storage
- All DB access goes through a `Store` interface
- Handler tests use `httptest.NewRecorder`
- Watch out: Chi middleware order matters -- auth before logging

Last activity (2026-03-05): Started phase 2, API routes scaffolded.
