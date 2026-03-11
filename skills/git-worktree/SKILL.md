---
name: git-worktree
description: >
  Manage git worktrees with a consistent repo layout: <repo>/src (main checkout) + <repo>/<branch>
  (worktrees). Use this skill whenever the user wants to clone a repo for worktree-based development,
  add a new worktree/branch, clean up worktrees, or asks about their worktree setup. Trigger on:
  "set up repo", "clone for worktrees", "new worktree", "add branch", "clean up worktrees",
  "remove worktrees", "worktree list", or any mention of the <repo>/src layout convention.
  Also trigger when the user says "create_worktree" or references their worktree workflow.
---

# Git Worktree Manager

Manage repos using a `<repo>/src` + `<repo>/<branch>` worktree layout. All git operations go through
the `gh` CLI or `git` commands — never edit git internals directly.

## Repo layout convention

```
my-repo/
├── src/          ← main checkout (main or master branch)
├── feature-x/    ← worktree for feature-x branch
├── bugfix-y/     ← worktree for bugfix-y branch
└── memory.md     ← shared project memory (managed by worktree-memory skill)
```

`src/` is always the primary checkout on the default branch (main or master). Worktrees sit as
siblings to `src/`, named after their branch. This layout keeps all branches of a project in one
directory and makes it easy to switch between them.

## Operations

### 1. Setup repo

Clone a repo and establish the worktree layout. Optionally create initial worktrees.

```bash
# Clone into <repo>/src
gh repo clone <owner/repo> <repo-name>/src

# Detect default branch
DEFAULT_BRANCH=$(git -C <repo-name>/src branch --show-current)

# Add upstream remote (for fork-based workflows)
# Skip if origin already points to upstream or user doesn't use forks
git -C <repo-name>/src remote add upstream <upstream-url>
git -C <repo-name>/src fetch upstream --prune

# Optionally create worktrees for existing branches (convert / to - for dir name)
DIR_NAME="${BRANCH//\//-}"
git -C <repo-name>/src worktree add ../$DIR_NAME <branch>

# Or create new branches from default
git -C <repo-name>/src worktree add -b <branch> ../$DIR_NAME
```

**Steps:**
1. Ask for the repo (owner/repo format) if not provided
2. Clone with `gh repo clone <owner/repo> <repo-name>/src`
3. Detect whether default branch is `main` or `master`
4. If the user works with forks, add the upstream remote
5. Create any requested worktrees

### 2. Add worktree

Create a new worktree from the current repo. Always fetch and update src before branching — this
ensures new branches start from the latest code and avoids drift.

```bash
SRC_DIR="<repo-name>/src"
BRANCH="<branch>"
DIR_NAME="${BRANCH//\//-}"  # Convert slashes to dashes for directory name

# Update src first — fetch from upstream (or origin if no upstream)
REMOTE=$(git -C "$SRC_DIR" remote | grep -q upstream && echo upstream || echo origin)
git -C "$SRC_DIR" fetch "$REMOTE" --prune

# Pull latest into src if it's clean
git -C "$SRC_DIR" pull --ff-only "$REMOTE" $(git -C "$SRC_DIR" branch --show-current) 2>/dev/null

# Check if branch exists on remote
if git -C "$SRC_DIR" ls-remote --exit-code --heads origin "$BRANCH" &>/dev/null; then
  # Checkout existing remote branch
  git -C "$SRC_DIR" worktree add "../$DIR_NAME" "$BRANCH"
else
  # Create new branch from base (default: main/master)
  git -C "$SRC_DIR" worktree add -b "$BRANCH" "../$DIR_NAME" "$REMOTE/<base>"
fi
```

The directory name converts `/` to `-` so `feature/auth` becomes `<repo>/feature-auth/` — keeping
all worktrees as flat siblings of `src/` and avoiding nested directory issues.

**Steps:**
1. Determine the repo — look for `src/` in the current directory or parent
2. Convert branch name slashes to dashes for the directory name
3. Fetch and pull latest into src (ff-only to avoid merge commits)
4. Check if branch already exists on the remote
5. If yes: check out existing branch as worktree
6. If no: create new branch from the specified base (defaults to default branch)
7. Report the worktree path

**Arguments:**
- `branch` (required) — branch name for the worktree
- `base` (optional) — base branch to create from, defaults to default branch (main/master)

### 3. Cleanup repo

Remove all worktrees and update src.

```bash
SRC_DIR="<repo-name>/src"

# List and remove all worktrees except src
git -C "$SRC_DIR" worktree list --porcelain \
  | grep "^worktree" \
  | awk '{print $2}' \
  | grep -v "$(cd "$SRC_DIR" && pwd)" \
  | while read -r wt; do
      echo "Removing worktree: $wt"
      git -C "$SRC_DIR" worktree remove --force "$wt"
    done

# Prune stale refs
git -C "$SRC_DIR" worktree prune

# Update src
REMOTE=$(git -C "$SRC_DIR" remote | grep -q upstream && echo upstream || echo origin)
git -C "$SRC_DIR" pull --ff-only "$REMOTE" $(git -C "$SRC_DIR" branch --show-current)
```

**Steps:**
1. Confirm with the user before removing worktrees (they may have uncommitted work)
2. List all worktrees, remove everything except src
3. Prune stale worktree references
4. Pull latest into src

### 4. List worktrees

Show current worktree status for a repo.

```bash
git -C "<repo-name>/src" worktree list
```

## Finding the repo context

When the user doesn't specify a repo name, detect it from the working directory:

1. If cwd contains a `src/` directory with a `.git` → cwd is the repo root
2. If cwd itself is inside a git worktree → find the main worktree, its parent is the repo root
3. Otherwise, ask the user

```bash
# From inside any worktree, find repo root
MAIN_WT=$(git worktree list | head -1 | awk '{print $1}')
REPO_ROOT=$(dirname "$MAIN_WT")
```

## Guidelines

- Always use `gh` CLI for cloning (`gh repo clone`) — it handles auth and fork detection
- Prefer `upstream` remote when available (fork workflows), fall back to `origin`
- Always fetch + pull src before creating new worktrees — stale bases cause merge pain later
- Use `--ff-only` for pulls to avoid accidental merge commits in src
- Confirm before cleanup — worktrees may contain uncommitted work
- Worktree directory names convert `/` to `-` (e.g., `feature/auth` → `feature-auth/`) to keep a flat layout
