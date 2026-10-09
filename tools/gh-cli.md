# GitHub CLI (`gh`) Cheat Sheet

Quick reference for the most common `gh` operations in this workflow. `gh` is the official GitHub CLI and is preferred over the GitHub web UI or MCP server for scripting and day-to-day repo work.

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

# View repo in browser
gh repo view --web

# List your repos
gh repo list owner --limit 100

# Create a new repo
gh repo create my-new-repo --public --source=.
```

---

## Pull Requests

### Create

```bash
# Create a draft PR with title and body from stdin
gh pr create --draft --title "fix: correct the thing" --body "Description here"

# Create a PR for a specific repo and target branch
gh pr create --repo owner/repo --base main --head feature-branch
```

### View & Review

```bash
# View the current branch's PR
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

# View PR comments/reviews
gh pr view 42 --json comments,reviews
```

### Edit & State

```bash
# Mark a PR as ready for review
gh pr ready 42

# Convert back to draft
gh pr ready 42 --undo

# Update PR title or body
gh pr edit 42 --title "new title" --body "new body"

# Edit body from a file
gh pr edit 42 --body-file pr-body.md

# Close a PR
gh pr close 42

# Reopen a PR
gh pr reopen 42
```

### Merge

```bash
# Squash merge and delete the remote branch
gh pr merge 42 --squash --delete-branch

# Merge with a merge commit
gh pr merge 42 --merge --delete-branch

# Rebase merge
gh pr merge 42 --rebase --delete-branch

# Auto-merge when checks pass
gh pr merge 42 --squash --auto
```

> The `--repo owner/repo` flag can be added to any `gh pr` command when not inside the repo directory.
>
> Example: `gh pr merge 2 --squash --delete-branch --repo VibsWorld/SwayamExamRegistration`

---

## Issues

```bash
# List open issues
gh issue list

# View an issue
gh issue view 7

# Create an issue
gh issue create --title "Bug: ..." --body "Description"

# Close an issue
gh issue close 7
```

---

## GitHub API Calls

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
| `--repo owner/repo` | Target a specific repository |
| `--web` | Open the result in a browser |
| `--json <fields>` | Output JSON for scripting |
| `--jq '<filter>'` | Filter JSON output with jq |
| `--silent` | Suppress non-error output |

---

## Tips

- Run `gh <command> --help` for the full option list of any command.
- Use `--jq` to build small one-liners instead of parsing plain text.
- Prefer `gh pr create --draft` for new PRs so the author can review before requesting reviews.
- Use `gh pr checks` before merging to confirm CI is green.
