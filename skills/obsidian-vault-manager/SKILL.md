---
name: obsidian-vault-manager
description: >
  ALWAYS use this skill when the user mentions Obsidian, their vault, or the
  `obsidian` CLI — even in passing. This skill manages notes, tasks, architecture
  decisions, and project context in the user's Obsidian vault via the `obsidian`
  CLI tool. Also trigger when the user asks to create meeting notes, log decisions,
  track tasks in their vault, update daily notes, check project status from notes,
  look up past decisions or ADRs, or search their knowledge base — these all live
  in Obsidian. Key phrases: "obsidian", "my vault", "vault notes", "daily note",
  "check my notes", "log a decision", "ADR", "what did we decide", "project status".
  If the user references their personal knowledge base, note-taking system, or
  persistent project documentation, this is likely the right skill.
---

# Obsidian Vault Manager

Interact with Obsidian vaults via the `obsidian` CLI to create project notes, track tasks, and use vault contents as persistent technical context.

## Why this skill exists

Obsidian vaults are markdown-based knowledge bases that persist across conversations. By reading and writing to the vault, you give the user a durable memory — project decisions, task status, meeting outcomes, and technical context all live in one place and carry forward between sessions.

## Vault Structure

Organize notes into these top-level folders:

```
Projects/          → Project docs, ADRs, feature specs
  {project-name}/
    README.md      → Project overview, goals, key links
    decisions/     → Architecture Decision Records
    notes/         → Meeting notes, research, scratchpad
Tasks/             → Task lists and tracking
  {project-name}.md → Tasks grouped by project
Daily/             → Daily notes (managed by Obsidian daily note plugin)
```

## Obsidian CLI Reference

The `obsidian` CLI talks directly to the running Obsidian app.

### Reading & Searching

```bash
# Search the vault
obsidian search query="auth migration" format=json

# Read a specific note
obsidian read                          # current file
obsidian files sort=modified limit=10  # recent files

# Check tags
obsidian tags counts

# Find broken links
obsidian unresolved
```

### Creating & Writing

```bash
# Create a note (uses vault templates if available)
obsidian create name="Projects/my-app/decisions/001-use-postgres" template=adr

# Append to daily note
obsidian daily:append content="## Meeting: Auth Migration\n- Decided on JWT approach\n- @alice owns implementation\n- Due: 2026-03-20"

# Open today's daily note
obsidian daily
```

### Tasks

```bash
# List tasks from a note
obsidian tasks daily
```

## Workflows

### Creating a Project Note

When the user asks to create a project note, meeting note, ADR, or any documentation:

1. Search the vault first to avoid duplicates: `obsidian search query="<topic>"`
2. Determine the right location based on note type:
   - **Meeting note** → `Projects/{project}/notes/YYYY-MM-DD-{topic}.md`
   - **ADR** → `Projects/{project}/decisions/NNN-{title}.md`
   - **Feature spec** → `Projects/{project}/notes/{feature-name}.md`
   - **General** → `Projects/{project}/notes/{title}.md`
3. Create with `obsidian create name="<path>"`
4. If a template exists for the note type, use `template=<name>`

#### Note Formats

**Meeting Note:**
```markdown
---
type: meeting
date: {{date}}
attendees: []
tags: [meeting, {project}]
---

# Meeting: {Title}

## Agenda
-

## Discussion
-

## Action Items
- [ ] {task} @due({date}) #{project}
```

**Architecture Decision Record:**
```markdown
---
type: adr
date: {{date}}
status: proposed
tags: [adr, {project}]
---

# ADR-{NNN}: {Title}

## Status
Proposed

## Context
{Why is this decision needed?}

## Decision
{What was decided}

## Consequences
{What follows from this decision}
```

### Task Tracking

Tasks use the Obsidian Tasks plugin format so they're queryable in the app:

```markdown
- [ ] Implement auth middleware @due(2026-03-15) #backend
- [x] Set up database schema ✅ 2026-03-08 #backend
- [ ] Write API docs @due(2026-03-20) #docs
```

When the user asks to add a task:
1. Determine the project → find or create `Tasks/{project}.md`
2. Append the task with appropriate tags, due date if mentioned
3. Also consider appending to the daily note for visibility

When the user asks about task status:
1. Search for tasks: `obsidian search query="- [ ]"` or check specific project task files
2. Read the relevant task files
3. Summarize: what's done, what's pending, what's overdue

### Project Status Check

When the user asks "what's the status of my project" or similar:

1. Search for the project folder: `obsidian files sort=modified limit=20`
2. Read the project README and recent notes
3. Check task files for pending/completed items
4. Summarize:
   - Recent activity (what changed lately)
   - Open tasks and upcoming due dates
   - Recent decisions
   - Any blockers or unresolved items from notes

### Using Vault as Technical Context

Before starting technical work, check the vault for relevant context:

1. Search for related notes: `obsidian search query="{topic}"`
2. Read any ADRs, specs, or notes that inform the current task
3. Reference what you found — "Based on ADR-003, you decided to use Postgres for this..."

This is especially valuable at the start of a new conversation. The vault bridges the gap between sessions.

After completing significant work, offer to log decisions or outcomes back to the vault so future sessions benefit.

### Daily Notes

Use daily notes as an activity log:

```bash
# Append a summary of what was accomplished
obsidian daily:append content="## Dev Session\n- Implemented auth middleware\n- Fixed #123 pagination bug\n- Decision: switching to Redis for session store (see ADR-005)"
```

Offer to update the daily note after completing tasks or making decisions.

## Guidelines

- Always search before creating to avoid duplicates
- Use consistent naming: lowercase-kebab-case for file paths
- Tag notes with project names for cross-referencing
- When reading vault context, summarize what you found — don't dump raw content
- Respect existing vault structure — if the user already has conventions, follow them
- Keep task descriptions short and actionable
- Include due dates and project tags on tasks when the information is available
