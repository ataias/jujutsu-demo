# Jujutsu Demo - Stacked PRs Workflow

This demo showcases Jujutsu (jj) features for managing stacked PRs.

## Prerequisites

```bash
# Ensure jj is installed
jj --version

# Configure your identity if not already done
jj config set --user user.name "Your Name"
jj config set --user user.email "your@email.com"
```

---

## Git Colocation

This repo uses **git colocation** - jj and git share the same `.git` directory.

### What is colocation?

```bash
# This repo was initialized with:
jj git init

# NOT with:
jj init  # This would create a pure jj repo without git
```

### Benefits of colocation:
- **Gradual adoption**: Use git commands when needed, jj when preferred
- **Tool compatibility**: IDEs, CI/CD, and other git tools work normally
- **Easy collaboration**: Push/pull to GitHub, GitLab, etc.
- **Escape hatch**: If jj confuses you, fall back to git

### How it works:
- jj stores its data in `.jj/` directory
- git data lives in `.git/` as usual
- jj automatically syncs with git on every operation
- `git_head()` in jj log shows the current git HEAD

### Using git alongside jj:
```bash
# You can still use git commands
git status
git log --oneline

# After using git directly, sync jj
jj git import

# jj automatically exports to git, but you can force it
jj git export
```

### Caveats:
- Avoid mixing jj and git operations on the same branch simultaneously
- Let jj manage branches (bookmarks) when possible
- If things get out of sync: `jj git import` usually fixes it

---

## Section 1: Basics - Creating and Navigating Changes

### View the current state
```bash
jj log
jj status
```

### Create a new change and describe it
```bash
# Make edits to app.js - add localStorage persistence
jj describe -m "Add localStorage persistence for tasks"
```

### View your changes
```bash
jj diff
jj show @
jj show --stat @
```

---

## Section 2: Git-like Index with `jj restore`

This shows how to selectively keep/discard changes (similar to git add -p / git stash).

### Scenario: You made changes to multiple files but only want to commit some

```bash
# First, make changes to BOTH styles.css and app.js
# Edit styles.css - change background color
# Edit app.js - add a new feature

# See all changes
jj diff

# OPTION A: Keep only specific file changes, discard the rest
jj restore styles.css  # Restores styles.css to parent version

# OPTION B: Restore from a specific revision
jj restore --from main styles.css

# OPTION C: Restore everything except one file (split workflow)
# First create a new change, then restore what you don't want
jj new
jj squash --interactive  # Interactive mode to select hunks
```

### Split changes into multiple commits
```bash
# If you have multiple logical changes in one commit:
jj split  # Opens editor to select which changes go in first commit
```

---

## Section 3: Stacked Changes with Bookmarks

### Create a stack of dependent changes

```bash
# Change 1: Add dark mode support
jj describe -m "PR1: Add dark mode CSS variables"
jj bookmark create -r @ feature/dark-mode-css

jj new
# Make more changes...
jj describe -m "PR2: Add dark mode toggle button"
jj bookmark create -r @ feature/dark-mode-toggle

jj new
# Make more changes...
jj describe -m "PR3: Persist dark mode preference"
jj bookmark create -r @ feature/dark-mode-persist
```

### View your stack
```bash
jj log
```

### Navigate the stack
```bash
# Jump to a specific change
jj edit feature/dark-mode-toggle

# Create a new change on top of current
jj new

# Go back to the top of the stack
jj edit feature/dark-mode-persist
```

---

## Section 4: Rebasing Stacked Changes

### Rebase a single change
```bash
jj rebase -r <change-id> --onto main
```

### Rebase all your tracked bookmarks at once (THE POWER MOVE)
```bash
# This rebases all heads of your bookmarked branches onto main
jj rebase -s 'heads(bookmarks() & mine() & ~(bookmarks() & mine())+ & ~main)' -d main@origin
```

#### Breaking down the revset:
- `bookmarks()` - all bookmarked changes
- `mine()` - changes authored by you
- `bookmarks() & mine()` - your bookmarked changes
- `~(bookmarks() & mine())+` - exclude descendants of your bookmarks
- `~main` - exclude main branch
- `heads(...)` - only the head commits (tips of branches)

### After rebasing, check for conflicts
```bash
jj log
jj status  # Will show if there are conflicts
jj resolve  # If there are conflicts
```

---

## Section 5: Pushing Multiple Branches

### Preview what will be pushed
```bash
jj git push --all --deleted --dry-run
```

### Push all bookmarks
```bash
jj git push --all --deleted
```

### Push specific bookmark
```bash
jj git push -b feature/dark-mode-css
```

### Creating PRs with gh CLI
```bash
# Create PR for current bookmark
gh pr create --title "PR1: Add dark mode CSS variables" --body "First PR in the stack"

# Create PR with specific base (for stacked PRs)
gh pr create --base feature/dark-mode-css --title "PR2: Add dark mode toggle"
```

---

## Section 6: Git Town Action - Visualizing Stacked PRs

This repo includes the [git-town/action](https://github.com/git-town/action) GitHub Action that automatically visualizes your PR stack.

### What it does:
- Adds a visual diagram to your PR description showing the stack
- Shows where the current PR sits in the stack
- Provides navigation links between related PRs

### How it works:
The workflow in `.github/workflows/git-town.yml` runs on every PR:

```yaml
name: Git Town

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  git-town:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: git-town/action@v1
        with:
          skip-single-stacks: true
```

### Configuration options:
- `skip-single-stacks: true` - Only shows visualization for actual stacks (2+ PRs)
- `location: comment` - Put visualization in a comment instead of PR description
- `history-limit: 100` - Limit PR history fetch for large repos

### Example visualization:
When you have stacked PRs, you'll see something like:
```
PR Stack:
  └── #3 PR3: Persist dark mode preference  ← You are here
      └── #2 PR2: Add dark mode toggle button
          └── #1 PR1: Add dark mode CSS variables
              └── main
```

---

## Section 7: The Undo Safety Net

### Made a mistake? No problem!
```bash
# Undo the last operation
jj undo

# See operation history
jj op log

# Restore to a specific operation
jj op restore <operation-id>
```

### Examples of undoable operations:
- Accidental rebase
- Wrong commit message
- Deleted bookmark
- Squashed wrong changes

---

## Section 8: Squashing Changes

### Squash current change into parent
```bash
jj squash
```

### Squash into a specific revision
```bash
jj squash --to <revision>
```

### Interactive squash (select specific changes)
```bash
jj squash --interactive
```

---

## Quick Reference Card

| Action | Command |
|--------|---------|
| View log | `jj log` |
| View status | `jj status` |
| View diff | `jj diff` |
| New change | `jj new` |
| Describe change | `jj describe -m "message"` |
| Edit existing change | `jj edit <rev>` |
| Create bookmark | `jj bookmark create -r @ name` |
| Delete bookmark | `jj bookmark delete name` |
| Restore file | `jj restore <path>` |
| Restore from rev | `jj restore --from <rev> <path>` |
| Rebase single | `jj rebase -r <rev> -d <dest>` |
| Rebase all bookmarks | `jj rebase -s 'heads(bookmarks() & mine() & ~(bookmarks() & mine())+ & ~main)' -d main@origin` |
| Push all | `jj git push --all --deleted` |
| Undo | `jj undo` |
| Squash | `jj squash` |
| Squash to rev | `jj squash --to <rev>` |
| Abandon change | `jj abandon` |
| Fetch | `jj git fetch` |

---

## Related Tools

### jj-spr (Super Pull Requests)

[jj-spr](https://github.com/LucioFranco/jj-spr) is a CLI tool that integrates Jujutsu with GitHub PRs more deeply.

**The problem it solves:**
Jujutsu encourages amending changes freely, but GitHub's review interface breaks when you force-push (reviewers lose context, diff history becomes confusing).

**How it works:**
- Maintains an "append-only PR branch" on GitHub
- You amend locally with `jj squash`, `jj describe`, etc.
- Updates create new commits on GitHub, preserving review context
- Everything squashes into one clean commit when merged

**Key commands:**
```bash
jj spr diff   # Create/update PRs
jj spr land   # Merge approved PRs
jj spr list   # View open PRs
jj spr close  # Close a PR
```

**Features:**
- Clean incremental diffs for reviewers
- Stacked PR support with automatic handling
- Land PRs in any order (cherry-pick mode)
- Change IDs survive rebases

---

## Demo Reset

To reset the demo to initial state:
```bash
jj abandon $(jj log -r 'all() & ~main' --no-graph -T 'change_id ++ " "')
jj edit main
jj new
```
