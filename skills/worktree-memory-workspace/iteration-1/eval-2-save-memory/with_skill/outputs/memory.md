# Testproject Memory

## Project
A test project for evaluating worktree memory skill.

## Current Phase
Phase 3: Caching — complete

## Key Decisions
- Go + Chi router: lightweight, stdlib-compatible
- SQLite for storage: simple, no external deps
- Redis for caching: chose over in-memory for persistence and shared access across instances

## Patterns
- Handler tests use httptest.NewRecorder
- All DB access through Store interface

## Gotchas
- Chi middleware order matters — auth before logging

## Log
- 2026-03-01: Project initialized, phase 1 complete
- 2026-03-05: Started phase 2, API routes scaffolded
- 2026-03-09: Phase 3 complete, decided Redis for caching over in-memory
