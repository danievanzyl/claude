---
name: worktree-memory
description: >
  Manage shared project memory across git worktrees. Use this skill at the START of every session
  to load project context, and after completing significant tasks to save decisions, patterns, and
  phase progress. Triggers on: session start, "save memory", "update memory", "sync memory",
  "load memory", "what phase are we on", or any mention of project memory/context. Also use this
  when the user asks to remember something or track project phases. This skill should activate
  whenever working in a git worktree — the memory file lives in the parent directory so all
  worktrees of the same project share context.
---

# Worktree Memory

Shared memory for git projects that persists across worktrees and sessions.

## Why this matters

Git worktrees create parallel working directories for branches, but Claude Code's built-in memory
is path-based — each worktree gets isolated memory. This skill resolves that by storing a single
`memory.md` in the worktree parent directory, so all branches of a project share context about
decisions, patterns, phases, and gotchas.

## Locating the memory file

Run these commands to find the correct memory path:

```bash
# Get the main worktree path (always first in the list)
MAIN_WORKTREE=$(git worktree list | head -1 | awk '{print $1}')

# Memory lives in the parent of the main worktree
MEMORY_DIR=$(dirname "$MAIN_WORKTREE")
MEMORY_FILE="$MEMORY_DIR/memory.md"
```

This works from any worktree of the project — `git worktree list` always returns the same list
regardless of which worktree you're in.

If `git worktree list` fails (not a git repo), fall back to `../memory.md` relative to the
current working directory.

## Session start — load memory

At the beginning of every session:

1. Run the commands above to locate `memory.md`
2. Read it if it exists
3. Briefly note the current phase/context to yourself (don't dump the whole file to the user)
4. If no memory file exists, that's fine — one will be created when there's something worth saving

## Saving memory — what to write

After completing a significant task, update `memory.md` with:

- **Decisions made and why** — architectural choices, tool selections, trade-offs
- **Patterns established** — conventions, naming, file structure
- **Gotchas or constraints** — things that broke, non-obvious requirements
- **Phase progress** — what phase the project is in, what's done, what's next

### Format

```markdown
# <Project Name> Memory

## Project
Brief one-liner about what this project is.

## Current Phase
Phase N: <name> — <status>

## Key Decisions
- <decision>: <why>

## Patterns
- <pattern>

## Gotchas
- <gotcha>

## Log
- YYYY-MM-DD: <what happened>
```

### Rules

- Keep it concise — this is a reference doc, not a journal
- Update existing sections rather than appending duplicates
- Log entries are append-only and chronological
- Use the project name from the git remote (e.g., `ferrix` from `github.com/user/ferrix.git`)
- Date entries using the current date

## Deriving project name

```bash
# Extract project name from git remote
git remote get-url origin 2>/dev/null | sed 's/.*\///' | sed 's/\.git$//'
```

If no remote exists, use the parent directory name of the main worktree.
