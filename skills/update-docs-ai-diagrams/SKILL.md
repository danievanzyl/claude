---
name: update-docs-ai-diagrams
description: Update Mermaid diagrams in documentation to reflect recent codebase changes.
allowed-tools: Bash(git *), Read, Grep, Glob, Edit, Write
---

# Update Mermaid Diagrams Skill

Analyze recent code changes and update any Mermaid diagrams in the project documentation so they stay accurate and current.

## Instructions

### 1. Identify Recent Changes

- Run `git diff main...HEAD --name-status` (or `git diff HEAD~5 --name-status` if on main) to list recently added, modified, or deleted files.
- Run `git log --oneline -10` for context on what changed.
- Focus on structural changes: new modules, services, endpoints, components, data models, workflows, or dependency changes.

### 2. Find Existing Mermaid Diagrams

- Search for Mermaid code blocks across the repo:
  - Glob for `**/*.md` and Grep for ` ```mermaid` in those files.
  - Also check for `.mmd` or `.mermaid` files via Glob `**/*.{mmd,mermaid}`.
- Read each file containing a diagram to understand what it documents.

### 3. Determine What Needs Updating

For each diagram, classify its type and check relevance to the recent changes:

| Diagram Type | Update When |
|---|---|
| `graph` / `flowchart` | New modules, services, or control flow paths added/removed |
| `sequenceDiagram` | API interactions, message flows, or service calls changed |
| `classDiagram` | Classes, interfaces, or inheritance hierarchies modified |
| `erDiagram` | Database models, entities, or relationships changed |
| `stateDiagram` | State machines or lifecycle transitions modified |
| `C4Context` / `C4Container` | System architecture or container boundaries changed |

Skip diagrams that are unaffected by the recent changes.

### 4. Update the Diagrams

- Read the relevant source code to understand the new structure accurately.
- Edit diagrams in place — preserve the existing style, direction, and formatting conventions.
- Add new nodes/edges for additions. Remove nodes/edges for deletions. Rename for refactors.
- Keep labels concise and consistent with the codebase naming.
- Do NOT rewrite diagrams from scratch unless the existing one is fundamentally wrong.

### 5. Validate

- Ensure every Mermaid block has matching opening and closing fences.
- Confirm node/edge names don't contain unescaped special characters that break Mermaid syntax.
- Verify no orphan nodes exist (every node should connect to at least one other).

### 6. Report

Provide a summary of changes made:

```
## Diagram Updates

- **<file>**: <what changed and why>
- **<file>**: No changes needed (not affected)
```

## Rules

- Only update diagrams that exist — do not create new diagram files unless explicitly asked.
- If no diagrams exist in the repo, report that and suggest where one could be added.
- Never fabricate components — every node in a diagram must correspond to real code.
- Preserve any custom styling (`style`, `classDef`, `linkStyle`) already in the diagram.
