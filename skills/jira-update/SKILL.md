---
name: jira-update
description: >-
  Post a concise development update to a Jira ticket summarising committed changes vs main,
  with optional status transition. Use this skill whenever the user says "/jira-update",
  "update the ticket with changes", "post changes to Jira", or wants to sync their git
  work back to a Jira ticket. Trigger format: /jira-update <ticket> [done|in progress|blocked].
  If only a ticket is given, posts a summary comment. If a status is also given, transitions
  the ticket after commenting.
allowed-tools: Bash(acli *), Bash(git *), Bash(gh *), Bash(cat *), Bash(wc *), Read, Write, Grep, Glob
---

# Jira Update

Post a condensed summary of committed changes to a Jira ticket and optionally transition its status.

## Usage

```
/jira-update <ticket> [status]
```

- `ticket` — Jira key like `CPE-1234`. If omitted, infer from branch name using pattern `[A-Z]+-\d+`.
- `status` — optional: `done`, `in progress`, or `blocked`. Maps to Jira statuses below.

## Workflow

### 1. Parse arguments

Extract `ticket` and optional `status` from the skill arguments.

Status mapping (case-insensitive):
| Argument     | Jira Status      |
|--------------|------------------|
| done         | DONE / RELEASED  |
| in progress  | In Progress      |
| blocked      | BLOCKED          |

If no ticket is provided, infer from branch:
```bash
git rev-parse --abbrev-ref HEAD
```
Extract `[A-Z]+-\d+`. If no match, ask the user — never guess.

### 2. Gather context

Run these in parallel:

**Commits vs main:**
```bash
git log main..HEAD --oneline
```

**Diff stat vs main:**
```bash
git diff main..HEAD --stat
```

**PR info (if exists):**
```bash
gh pr list --head "$(git rev-parse --abbrev-ref HEAD)" --json number,title,url
```

**Detect gotchas from the diff** — scan for:
- Migration files (DB schema changes)
- Config/env changes (.env, config/, settings files)
- Breaking API changes (removed endpoints, changed signatures)
- New dependencies (package.json, go.mod, requirements.txt, Cargo.toml changes)
- Security-sensitive changes (auth, permissions, secrets)

Only flag gotchas that are actually present. Don't fabricate warnings.

### 3. Build the comment

Write a temp file to avoid shell escaping issues. The comment should be **short** — the PR link is the source of truth, so don't repeat the full diff.

Format:
```
Development Update

<2-3 sentence TLDR of what changed and why>

[if gotchas exist]
Heads up: <one-liner per gotcha>

[if PR exists]
GitHub: <PR URL>
```

Keep the TLDR genuinely concise. Think "what would a PM need to know in 10 seconds". Don't list every file or commit — summarise the intent.

```bash
cat > /tmp/jira-update-comment.txt << 'COMMENT'
<built comment here>
COMMENT

acli jira workitem comment create --key <TICKET> --body-file /tmp/jira-update-comment.txt
```

### 4. Transition (if status provided)

Only if the user passed a status argument:

```bash
acli jira workitem transition --key <TICKET> --status "<mapped status>" --yes
```

Report success/failure to the user.

### 5. Confirm

Tell the user what was posted. Keep it to one line, e.g.:
```
Posted update to CPE-1234. Transitioned to In Progress.
```
or:
```
Posted update to CPE-1234.
```

## Rules

- Always use `--body-file` for comments — never inline multiline text in `--body`.
- Always use `--yes` on transitions.
- Always use `--json` on data-retrieval commands.
- If `acli` fails with auth errors (401, unauthorized, token expired), stop and tell the user to run `acli auth login`.
- Use ONLY `acli jira workitem` subcommands — no legacy commands.
- Don't over-explain in the Jira comment. The PR is the detailed record.
