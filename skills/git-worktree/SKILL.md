---
name: git-worktree
description: >
  Manage git worktrees with a consistent repo layout: <repo>/src (main checkout) + <repo>/<branch>
  (worktrees). Use this skill whenever the user wants to: start a new feature ("new feature X",
  "start feature X"), close/clean up a worktree after a PR is merged ("close worktree", "clean up
  branch X", "worktree done"), clone a repo for worktree-based development, add a new worktree/branch,
  list worktrees, or asks about their worktree setup. Also trigger on: "set up repo", "clone for
  worktrees", "new worktree", "add branch", "remove worktree", "worktree list", "create_worktree",
  or any mention of the <repo>/src layout convention.
---

# Git Worktree Manager

Manage repos using a `<repo>/src` + `<repo>/<branch>` worktree layout. All git operations go through
the `gh` CLI or `git` commands — never edit git internals directly.

## Repo layout convention

```
my-repo/
├── src/              ← main checkout (main or master branch)
├── feature-auth/     ← worktree for feature/auth branch
├── bugfix-y/         ← worktree for bugfix-y branch
└── memory.md         ← shared project memory (managed by worktree-memory skill)
```

`src/` is always the primary checkout on the default branch (main or master). Worktrees sit as
siblings to `src/`, named after their branch (with `/` converted to `-`). This layout keeps all
branches of a project in one directory and makes switching between them easy.

## Operations

### 1. New feature

Start a new feature branch and worktree. This is the most common operation — the user says something
like "new feature ticket-name" or "start feature auth-refactor".

**Branch naming**: Always use `feature/<name>` format. If the user gives just a name like "auth-refactor",
prefix it with `feature/`. If they already include `feature/`, use as-is.

```bash
SRC_DIR="<repo-name>/src"
BRANCH="feature/<name>"
DIR_NAME="${BRANCH//\//-}"  # feature/auth → feature-auth

# Update src first — always branch from latest main
REMOTE=$(git -C "$SRC_DIR" remote | grep -q upstream && echo upstream || echo origin)
DEFAULT_BRANCH=$(git -C "$SRC_DIR" branch --show-current)
git -C "$SRC_DIR" fetch "$REMOTE" --prune
git -C "$SRC_DIR" pull --ff-only "$REMOTE" "$DEFAULT_BRANCH" 2>/dev/null

# Create new branch from updated main
git -C "$SRC_DIR" worktree add -b "$BRANCH" "../$DIR_NAME" "$REMOTE/$DEFAULT_BRANCH"

# Change into the new worktree
cd "$REPO_ROOT/$DIR_NAME"
```

**Steps:**
1. Determine the repo root — look for `src/` in cwd or parent
2. Build branch name: `feature/<name>` (prefix if needed)
3. Convert branch to dir name: `feature/X` → `feature-X`
4. Fetch and pull latest into src (ff-only)
5. Create worktree with new branch from updated default branch
6. `cd` into the new worktree so subsequent commands run there

**If the user specifies a base branch** other than main/master, use that instead of the default branch.

### 2. Close worktree (after PR merge)

Remove a worktree after its PR has been merged. The user says "close worktree", "clean up branch X",
"worktree done", etc.

This operation is cautious — it checks the PR is actually merged before deleting anything.

```bash
SRC_DIR="<repo-name>/src"
BRANCH="<branch>"
DIR_NAME="${BRANCH//\//-}"
WT_PATH="<repo-root>/$DIR_NAME"

# Step 1: Check PR merge status
gh pr view "$BRANCH" --repo <owner/repo> --json state --jq '.state'
# Must return "MERGED". If not, warn the user and stop.

# Step 2: Check for uncommitted changes in the worktree
git -C "$WT_PATH" status --porcelain
# If dirty, warn the user and ask for confirmation before proceeding.

# Step 3: Remove the worktree
git -C "$SRC_DIR" worktree remove --force "$WT_PATH"

# Step 4: Delete the local branch
git -C "$SRC_DIR" branch -d "$BRANCH"
# Use -d (not -D) so git refuses if branch isn't fully merged.
# If -d fails, tell the user and let them decide whether to force-delete.

# Step 5: Prune stale refs
git -C "$SRC_DIR" worktree prune
```

**Steps:**
1. Determine the branch — from args, or detect from cwd if inside a worktree
2. Check PR status via `gh pr view`. If not merged, tell the user and stop
3. Check for uncommitted changes. If dirty, warn and ask for confirmation
4. Remove the worktree via `git worktree remove`
5. Delete the local branch with `git branch -d`
6. Prune stale worktree refs
7. Confirm cleanup is complete

**Detecting the branch from cwd**: If the user runs "close worktree" while inside one, detect it:
```bash
BRANCH=$(git branch --show-current)
```

**Detecting the repo**: Use `gh repo view --json nameWithOwner --jq '.nameWithOwner'` from the worktree
or src directory to get the `owner/repo` for the PR check.

### 3. Add worktree (non-feature)

For branches that don't follow the `feature/` convention (bugfixes, experiments, checking out
someone else's branch, etc.).

```bash
SRC_DIR="<repo-name>/src"
BRANCH="<branch>"
DIR_NAME="${BRANCH//\//-}"

# Update src
REMOTE=$(git -C "$SRC_DIR" remote | grep -q upstream && echo upstream || echo origin)
git -C "$SRC_DIR" fetch "$REMOTE" --prune
git -C "$SRC_DIR" pull --ff-only "$REMOTE" $(git -C "$SRC_DIR" branch --show-current) 2>/dev/null

# Check if branch exists on remote
if git -C "$SRC_DIR" ls-remote --exit-code --heads origin "$BRANCH" &>/dev/null; then
  git -C "$SRC_DIR" worktree add "../$DIR_NAME" "$BRANCH"
else
  git -C "$SRC_DIR" worktree add -b "$BRANCH" "../$DIR_NAME" "$REMOTE/<base>"
fi
```

**Steps:**
1. Determine the repo root
2. Convert branch name slashes to dashes for directory name
3. Fetch and pull latest into src
4. If branch exists on remote: check out as worktree
5. If not: create new branch from specified base (defaults to default branch)
6. Report the worktree path

### 4. Setup repo

Clone a repo and establish the worktree layout.

```bash
gh repo clone <owner/repo> <repo-name>/src

DEFAULT_BRANCH=$(git -C <repo-name>/src branch --show-current)

# For fork workflows
git -C <repo-name>/src remote add upstream <upstream-url>
git -C <repo-name>/src fetch upstream --prune
```

**Steps:**
1. Ask for the repo (owner/repo format) if not provided
2. Clone with `gh repo clone <owner/repo> <repo-name>/src`
3. Detect default branch (main or master)
4. If fork workflow, add upstream remote

### 5. Cleanup all worktrees

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

git -C "$SRC_DIR" worktree prune

REMOTE=$(git -C "$SRC_DIR" remote | grep -q upstream && echo upstream || echo origin)
git -C "$SRC_DIR" pull --ff-only "$REMOTE" $(git -C "$SRC_DIR" branch --show-current)
```

**Steps:**
1. Confirm with the user before removing — worktrees may have uncommitted work
2. List all worktrees, remove everything except src
3. Prune stale refs
4. Pull latest into src

### 6. List worktrees

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

- Always use `gh` CLI for cloning and PR checks — handles auth automatically
- Prefer `upstream` remote when available (fork workflows), fall back to `origin`
- Always fetch + pull src before creating new worktrees — stale bases cause merge pain
- Use `--ff-only` for pulls to avoid accidental merge commits in src
- Confirm before destructive operations — worktrees may contain uncommitted work
- Directory names convert `/` to `-` (e.g., `feature/auth` → `feature-auth/`)
- Use `git branch -d` (not `-D`) when cleaning up — let git protect against data loss
