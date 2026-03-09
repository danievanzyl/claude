---
name: gh-github
description: >-
  Use the gh CLI to interact with GitHub when the user references GitHub URLs, repos, PRs, issues,
  actions, releases, or any github.com resource. Activates for GitHub URLs (github.com/*),
  PR/issue references (#123), repo references (owner/repo), or questions about GitHub resources.
allowed-tools: Bash(gh *), Read
---

# GitHub CLI Skill

Use `gh` CLI for all GitHub interactions. Never use raw API calls, `curl`, or `WebFetch` for github.com URLs.

## URL Parsing

When the user provides a GitHub URL, extract the relevant parts and map to `gh` subcommands:

| URL Pattern | Command |
|---|---|
| `github.com/OWNER/REPO` | `gh repo view OWNER/REPO` |
| `github.com/OWNER/REPO/pull/N` | `gh pr view N --repo OWNER/REPO` |
| `github.com/OWNER/REPO/issues/N` | `gh issue view N --repo OWNER/REPO` |
| `github.com/OWNER/REPO/actions/runs/ID` | `gh run view ID --repo OWNER/REPO` |
| `github.com/OWNER/REPO/releases/tag/TAG` | `gh release view TAG --repo OWNER/REPO` |
| `github.com/OWNER/REPO/commit/SHA` | `gh api repos/OWNER/REPO/commits/SHA` |
| `github.com/OWNER/REPO/blob/...` | `gh api` to fetch file contents or clone |

Always include `--repo OWNER/REPO` when operating outside the current repo context.

## Common Operations

### PRs
```bash
gh pr view N                    # view PR details
gh pr view N --comments         # view PR with comments
gh pr diff N                    # view PR diff
gh pr checks N                  # view CI status
gh pr list                      # list open PRs
gh pr list --state merged       # list merged PRs
gh pr list --author @me         # my PRs
gh pr review N --approve        # approve PR
gh pr merge N --squash          # merge PR
```

### Issues
```bash
gh issue view N                 # view issue
gh issue list                   # list open issues
gh issue list --label bug       # filter by label
gh issue create --title "..." --body "..."
gh issue comment N --body "..."
```

### Repos
```bash
gh repo view OWNER/REPO         # repo overview
gh repo clone OWNER/REPO        # clone
gh repo list OWNER              # list repos for owner
gh search repos "query"         # search repos
```

### Actions / CI
```bash
gh run list                     # recent workflow runs
gh run view ID                  # run details
gh run view ID --log            # full logs
gh run view ID --log-failed     # only failed step logs
gh run rerun ID                 # rerun a failed run
gh workflow list                # list workflows
gh workflow run NAME            # trigger workflow
```

### Releases
```bash
gh release list                 # list releases
gh release view TAG             # view specific release
gh release create TAG --generate-notes
```

### API (fallback for anything gh doesn't have a subcommand for)
```bash
gh api repos/OWNER/REPO/commits/SHA
gh api repos/OWNER/REPO/pulls/N/comments
gh api repos/OWNER/REPO/actions/artifacts
gh api graphql -f query='{ ... }'
```

## Rules

- Always prefer `gh` subcommands over `gh api` when a subcommand exists.
- For PR review comments use `gh api repos/OWNER/REPO/pulls/N/comments` (review comments) vs `gh issue view N --comments` (conversation comments).
- Use `--json` flag + `--jq` for extracting specific fields when the user needs structured data.
- If `gh auth status` fails, tell the user to run `gh auth login`.
- When inside a git repo that tracks a GitHub remote, `--repo` can be omitted.
- For large outputs (logs, diffs), pipe through `head` or use `--limit` flags to avoid flooding the terminal.
