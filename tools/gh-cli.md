# GitHub CLI (`gh`) Cheat Sheet

Quick reference for the most common `gh` operations in this workflow. `gh` is the official GitHub CLI and is **preferred over the GitHub MCP server** for all GitHub operations.

> Why `gh`? It is canonical, fully-supported, composes with shell tooling (`jq`, `grep`), and does not require MCP protocol permissions or token juggling. Use `gh api` for raw REST/GraphQL before falling back to any MCP server.
>
> See `D:/garima/CLAUDE.md` § *GitHub CLI (`gh`) — preferred over the GitHub MCP server* for the project rule.

---

## Authentication

```bash
# Log in (opens browser)
gh auth login

# Check current authentication state
gh auth status

# Log out
gh auth logout
```

---

## Repositories

```bash
# Clone a repo
gh repo clone owner/repo

# View repo details in terminal
gh repo view owner/repo

# View repo in browser
gh repo view owner/repo --web

# List repos for an owner
gh repo list owner --limit 100

# Create a new repo from the current directory
gh repo create my-new-repo --public --source=.
```

---

## Branches

```bash
# List branches
gh branch list

# Delete a branch
gh branch delete feature/old-branch
```

> Prefer plain `git checkout -b feature/name` locally; use `gh` for GitHub-hosted operations.

---

## Pull Requests

### Create

```bash
# Draft PR — always create as draft first (project rule)
gh pr create --draft --title "fix: correct the thing" --body "Description here"

# Create PR for a specific repo and target branch
gh pr create --repo owner/repo --base main --head feature-branch --title "fix: ..." --body "..."

# Create PR with body read from a file
gh pr create --draft --title "feat: add feature" --body-file pr-body.md
```

> Never create a PR without the user's explicit consent. Ask for a ticket ID before creating; if they provide one, add `Fixes: TICKET-123` to the PR description.

### View & Review

```bash
# View PR for the current branch
gh pr view

# View a specific PR by number
gh pr view 42

# Open PR in browser
gh pr view 42 --web

# List open PRs
gh pr list

# List PRs with filters
gh pr list --author @me --state open --limit 20
```

### Checks & Comments

```bash
# Watch CI checks for the current PR
gh pr checks

# View checks as JSON
gh pr checks 42 --json name,bucket,conclusion

# View PR comments and reviews
gh pr view 42 --json comments,reviews

# Reply to a review comment in the original inline thread
gh pr comment 42 --body $'[AI-Assisted Response]\n\nReply text here'

# Add a general PR comment
gh pr comment 42 --body "General comment text"
```

> When posting AI-assisted replies on GitHub/Linear, prepend `[AI-Assisted Response]` or `[AI-Assisted | Reviewed by Author]` and leave a blank line before the body.

### Edit & State

```bash
# Mark a PR as ready for review
gh pr ready 42

# Convert back to draft
gh pr ready 42 --undo

# Update PR title or body
gh pr edit 42 --title "new title" --body "new body"

# Update PR body from a file
gh pr edit 42 --body-file pr-body.md

# Close a PR
gh pr close 42

# Reopen a PR
gh pr reopen 42
```

### Merge

```bash
# Squash merge and delete the remote branch (preferred)
gh pr merge 42 --squash --delete-branch

# Merge a PR in another repo
gh pr merge 2 --squash --delete-branch --repo owner/repo

# Merge with a merge commit
gh pr merge 42 --merge --delete-branch

# Rebase merge
gh pr merge 42 --rebase --delete-branch

# Auto-merge when checks pass
gh pr merge 42 --squash --auto
```

> Only merge when the user is the PR author/requestor. Use **Squash and Merge**. Delete the feature branch after merge. Check `gh pr checks` first; all checks must be green.

---

## Issues

```bash
# List open issues
gh issue list

# View an issue
gh issue view 7

# Create an issue
gh issue create --title "Bug: ..." --body "Description"

# Create with body from a file
gh issue create --title "Bug: ..." --body-file issue-body.md

# Close an issue
gh issue close 7

# Reopen an issue
gh issue reopen 7
```

---

## GitHub API Calls (`gh api`)

Use `gh api` when the high-level commands don't expose the field you need.

```bash
# GET request
gh api repos/owner/repo/issues

# GET with jq filtering
gh api repos/owner/repo/issues --jq '.[] | {number, title}'

# POST request
gh api repos/owner/repo/issues -X POST --field title="Bug" --field body="Details"

# PUT file contents via Contents API
gh api repos/owner/repo/contents/path/to/file.md -X PUT \
  --field message="chore: update file" \
  --field branch="main" \
  --field sha="EXISTING_SHA" \
  --field content="$(base64 -w0 local-file.md)"
```

> For large files where `base64 -w0 local-file.md` hits argument-length limits, write the base64 to a temp file: `--field content=@/tmp/file.b64`.

### Common `gh api` patterns

```bash
# Get the SHA of a remote file (needed for updates)
gh api repos/owner/repo/contents/path/file.md --jq .sha

# List contents of a directory
gh api repos/owner/repo/contents/path --jq '.[] | {name, type, sha}'

# Get PR reviews and comments as JSON
gh api repos/owner/repo/pulls/42 --jq '{number, state, title, body}'
gh api repos/owner/repo/issues/42/comments --jq '.[] | {id, user: .user.login, body}'
```

---

## Releases

```bash
# List releases
gh release list

# Create a release from a tag
gh release create v1.2.3 --title "Version 1.2.3" --notes "Release notes"

# Generate release notes automatically
gh release create v1.2.3 --generate-notes
```

---

## Common Flags

| Flag | Meaning |
|------|---------|
| `--repo owner/repo` | Target a specific repository when not inside its directory |
| `--web` | Open the result in a browser |
| `--json <fields>` | Output JSON for scripting |
| `--jq '<filter>'` | Filter JSON output with jq |
| `--silent` | Suppress non-error output |
| `--paginate` | Follow pagination and return all pages |

---

## Workflow Checklist

Use `gh` for these common tasks instead of the web UI or MCP:

- [ ] Create a draft PR: `gh pr create --draft ...`
- [ ] Check CI before merge: `gh pr checks <n>`
- [ ] Mark PR ready: `gh pr ready <n>`
- [ ] Squash merge: `gh pr merge <n> --squash --delete-branch`
- [ ] Add PR description footer: `gh pr edit <n> --body-file file.md`
- [ ] Read/write remote files: `gh api repos/owner/repo/contents/...`
- [ ] List/filter issues and PRs: `gh issue list --jq ...`, `gh pr list --jq ...`

---

## Tips

- Run `gh <command> --help` for the full option list.
- Prefer `gh pr create --draft` so the author can review before requesting reviews.
- Use `gh pr checks` before merging; never merge with failing checks.
- Combine `gh api --jq` for quick inspection instead of parsing plain text.
- When using `--repo owner/repo`, you can run commands from any directory.
