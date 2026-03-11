---
name: jira-acli
description: >-
  Manage Jira tickets via the acli CLI. Use this skill whenever the user mentions Jira tickets,
  ticket status, sprint work, CPE tickets, or wants to interact with their Jira board — even
  if they don't say "Jira" explicitly. Triggers on: ticket references (CPE-1234), status changes
  ("move to done", "start working on"), listing assigned work, creating tickets or sub-tasks,
  linking PRs to tickets, or summarising PR changes for Jira. Also activate when the user says
  "my tickets", "what am I working on", "create a ticket", "update the ticket", or references
  a board/sprint.
allowed-tools: Bash(acli *), Bash(git *), Bash(gh *), Read, Grep, Glob
---

# Jira CLI Skill

Interact with Jira using `acli jira` commands. The primary project is **CPE** on board **96**.

## Critical: Command Format

All ticket operations go through `acli jira workitem <subcommand>`. Do NOT use legacy commands like `acli jira createSubtask`, `acli jira createIssue`, etc. — they do not exist.

The issue type for subtasks is `Subtask` (one word, no hyphen). NOT `Sub-task`, NOT `sub-task`.

Use ONLY the exact commands documented in this skill. Do not guess or improvise acli commands.

## Ticket Number Inference

When the user doesn't supply a ticket number, infer it from the current git branch:

```bash
git rev-parse --abbrev-ref HEAD
```

Extract the ticket key using the pattern `[A-Z]+-\d+` (e.g., `CPE-2272` from branch `CPE-2272-add-routes`). If no match is found, ask the user for the ticket number — never guess.

## Core Operations

### List My Tickets

```bash
# All tickets assigned to me
acli jira workitem search --jql "assignee = currentUser() AND project = CPE" \
  --fields "key,summary,status,priority" --json

# Only open tickets
acli jira workitem search \
  --jql 'assignee = currentUser() AND project = CPE AND status not in ("DONE / RELEASED")' \
  --fields "key,summary,status,priority" --json

# Current sprint tickets
acli jira sprint list-workitems --board 96 --sprint <SPRINT_ID> \
  --jql "assignee = currentUser()" --fields "key,summary,status,assignee" --json
```

To find the active sprint ID, list sprints first:
```bash
acli jira board list-sprints --board 96 --json
```
Pick the sprint with `state: "active"`.

### View Ticket Details

```bash
acli jira workitem view CPE-1234 --json
```

### Assign Ticket to Me

```bash
acli jira workitem assign --key CPE-1234 --assignee @me
```

### Change Ticket Status

The CPE board uses these statuses: **To Do**, **In Progress**, **BLOCKED**, **DONE / RELEASED**.

```bash
acli jira workitem transition --key CPE-1234 --status "In Progress" --yes
```

Statuses are board-dependent. If a transition fails, the target status may not be valid for that ticket's workflow. Try viewing the ticket to check its current status before transitioning.

### Create a New Ticket

```bash
acli jira workitem create \
  --project CPE \
  --type Task \
  --summary "Short description of the work" \
  --description "Detailed description of what needs to be done" \
  --assignee @me \
  --json
```

Valid types: **Task**, **Epic**, **Subtask**, **Story**, **Bug**, **Dependency**, **Critical**.

When creating, always confirm the summary and type with the user before running the command.

### Create a Sub-task

Sub-tasks require a `--parent` flag pointing to the parent ticket:

```bash
acli jira workitem create \
  --project CPE \
  --type Subtask \
  --parent CPE-1234 \
  --summary "Subtask description" \
  --assignee @me \
  --json
```

If the user says "create sub-tasks on CPE-1234", infer the parent from context or the current branch. For multiple sub-tasks, create them sequentially and report all created keys at the end.

### Add a Comment

```bash
acli jira workitem comment create --key CPE-1234 --body "Comment text here"
```

## PR Summary Workflow

When the user asks to summarise a PR for a ticket (or "update the ticket with PR info"):

1. **Get the ticket key** — from the branch or user input
2. **Get the PR details** — run:
   ```bash
   gh pr list --head "$(git rev-parse --abbrev-ref HEAD)" --json number,title,url,body
   ```
   If no PR is found for the current branch, ask the user for a PR number/URL.
3. **Get the diff summary** — run `gh pr diff <N> | head -200` to understand the changes
4. **Build the comment** — the comment MUST contain both a TLDR summary AND the GitHub PR URL. Use this exact format:
   ```
   PR #<number>: <title>

   <2-4 sentence TLDR of what the PR changes>

   GitHub: <full PR URL>
   ```
5. **Post the comment** — use `--body-file` to avoid shell escaping issues with the comment body:
   ```bash
   # Write comment to temp file first
   cat > /tmp/jira-comment.txt << 'COMMENT'
   PR #42: Add route configuration

   Adds VPN route tables for the on-prem network, configures BGP peering with the transit gateway, and updates security group rules to allow traffic on ports 443 and 8080.

   GitHub: https://github.com/org/repo/pull/42
   COMMENT

   acli jira workitem comment create --key CPE-1234 --body-file /tmp/jira-comment.txt
   ```

The GitHub PR link is essential — without it the comment is incomplete. Always verify the URL is present before posting.

## Formatting Output

When displaying ticket lists to the user, format as a readable table:

```
| Key      | Summary              | Status      |
|----------|----------------------|-------------|
| CPE-2665 | pkl-common           | In Progress |
| CPE-2640 | Update terraform ... | To Do       |
```

For single ticket views, show key fields: **Key**, **Summary**, **Status**, **Assignee**, **Type**, **Priority**, **Description** (truncated if long).

## JQL Gotchas

- Use single quotes around the full JQL string to avoid shell escaping issues.
- Never use `!=` with status values containing special characters (like `/`). Use `status not in ("DONE / RELEASED")` instead — JQL treats `\!` as an illegal escape sequence.
- For multi-word status names, always double-quote them inside the JQL: `status = "In Progress"`.

## Rules

- Always use `--json` output from acli and parse it — raw text output is unreliable for extraction.
- Always use `--yes` flag on transitions to avoid interactive prompts.
- Never transition a ticket without confirming the target status with the user first, unless the intent is unambiguous (e.g., "move CPE-1234 to done").
- When creating tickets, confirm summary/type/parent with the user before executing.
- If any `acli` command fails with an authentication or token error (e.g., `401`, `unauthorized`, `token expired`, `session expired`), stop immediately and tell the user: **"Your Jira auth has expired. Run `acli auth login` to reauthenticate, then try again."** Do not retry the command or attempt any other acli operations until the user confirms they've re-authenticated.
- Prefer `@me` over email addresses for assignment.
- ONLY use `acli jira workitem` subcommands. Never use `acli jira createSubtask`, `acli jira createIssue`, or any other top-level jira commands — they don't work.
- The subtask type is `Subtask` (one word). Never use `Sub-task`.
