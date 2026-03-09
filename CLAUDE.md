# IMPORTANT: Always read the whole contents of this file

- In all interactions and commit messages, be extremely concise and sacrifice grammar for the sake of concision.

## Memory Protocol

At the start of each session, check for a `memory.md` file in the parent directory of the current working directory and read it if it exists.
After completing any significant task, append a dated entry to `../memory.md` covering:

- Decisions made and why
- Patterns established
- Gotchas or constraints discovered

## Tool Preferences

- Use `fd` instead of `find`
- Use `rg` (ripgrep) instead of `grep`

## Plans

- At the end of each plan, give me a list of unresolved questions to answer, if any. Make the question extremely concise. Sacrifice grammar for the sake of concision.

## Github

- Your primary method for interacting with Github should be the Github CLI.

## Project Boundaries

- When searching for files or context, scope all commands to `.` (current project root). Do not use absolute paths or `../` traversal. If you need external context, ask first.

## Project Phase tracking

- Remember to keep track of phases in your memory
