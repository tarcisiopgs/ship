---
name: ship
description: Use this skill when the user wants to ship work via a Pull Request and automatically return the repository to its original clean state. Trigger when the user says "ship", "ship it", "ship this PR", "create a PR and cleanup", "done with this branch", "push and go back to main", "finalize this feature", or any variation of wanting to open a PR and restore the workspace — whether they're on a regular branch or inside a git worktree. Always use this skill instead of manually creating PRs when cleanup is expected afterward.
---

# Ship

Ship the current work via a Pull Request and restore the repository to its original state.

**Announce at start:** "I'm using the ship skill to create your PR and clean up the workspace."

## Overview

This skill is opinionated and streamlined: it always ships via PR and always restores the repository afterward. No menu, no options — just push, PR, and return to clean state.

The full flow:
1. Detect context (worktree or regular branch)
2. Handle uncommitted changes
3. Verify tests pass
4. Push and create PR
5. Restore original state

## Step 1: Detect Context

Run both commands and capture the output:

```bash
git worktree list
git rev-parse --show-toplevel
git branch --show-current
```

From `git worktree list`, the **first line** is always the main worktree.

- **Worktree context**: current path (`show-toplevel`) matches a line other than the first in `worktree list`
- **Regular branch context**: current path is the main worktree path (first line)

Capture and store:
- `CURRENT_BRANCH`: output of `git branch --show-current`
- `MAIN_WORKTREE_PATH`: first path from `git worktree list`
- `CURRENT_WORKTREE_PATH`: output of `git rev-parse --show-toplevel` (only relevant if in worktree context)
- `IS_WORKTREE`: boolean

## Step 2: Handle Uncommitted Changes

```bash
git status --short
```

If there are uncommitted changes, ask:

```
There are uncommitted changes. What would you like to do?

1. Commit them now (I'll write a commit message based on the changes)
2. Stash them
3. Cancel — I'll handle this myself

Which option?
```

- **Option 1**: Stage all and commit with a message derived from `git diff` summary
- **Option 2**: `git stash push -m "ship: stashed before PR"` — remind the user to pop stash later
- **Option 3**: Stop entirely

If there are no uncommitted changes, continue silently.

## Step 3: Verify Tests

Auto-detect the project's test suite and run it:

```bash
# Priority order — use the first match
if [ -f "bun.lockb" ] || grep -q '"test"' package.json 2>/dev/null; then
  bun test
elif [ -f "package.json" ]; then
  npm test
elif [ -f "Cargo.toml" ]; then
  cargo test
elif [ -f "pyproject.toml" ] || [ -f "requirements.txt" ]; then
  pytest
elif [ -f "go.mod" ]; then
  go test ./...
fi
```

**If tests fail:**
```
Tests failing before ship — cannot create PR with broken tests.

[paste failures]

Fix the failures and run /ship again.
```
Stop.

**If no test suite is detected:** Warn the user and ask whether to proceed without test verification.

**If tests pass:** Continue silently.

## Step 4: Determine Base Branch

```bash
# Try to detect the remote default branch
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')

# Fallback: check common names
if [ -z "$BASE" ]; then
  git fetch origin --quiet 2>/dev/null
  BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
fi

# Last resort fallback
if [ -z "$BASE" ]; then
  BASE="main"
fi
```

## Step 5: Push and Create PR

```bash
# Push the branch
git push -u origin "$CURRENT_BRANCH"
```

Build the PR title from the branch name: strip common prefixes (`feat/`, `fix/`, `feature/`, `chore/`, `refactor/`, `hotfix/`), replace hyphens with spaces, capitalize. Cross-reference with recent commits to make it human-readable.

Build the PR body from `git log $BASE..HEAD` and `git diff $BASE...HEAD --stat`:

```bash
gh pr create \
  --title "<derived title>" \
  --body "$(cat <<'EOF'
## Summary
<2-3 bullets summarizing what changed, derived from commits and diff>

## Test Plan
- [ ] <key verification steps based on what changed>

🤖 Shipped via [ship skill](https://github.com/tarcisiopgs/ship)
EOF
)"
```

Print the PR URL prominently after creation.

## Step 6: Restore Original State

### If in a Worktree

```bash
git worktree remove "$CURRENT_WORKTREE_PATH"
```

Since you cannot change the user's shell working directory, **always print explicit instructions**:

```
PR created: <URL>

Worktree removed. Switch back to your main workspace:

  cd <MAIN_WORKTREE_PATH>
```

### If on a Regular Branch

```bash
git checkout "$BASE"
git pull
```

Report:
```
PR created: <URL>
Switched to <base-branch> and pulled latest.
```

## Quick Reference

| Context | After PR created |
|---------|-----------------|
| Worktree | `git worktree remove <path>` + instruct `cd` to main |
| Regular branch | `git checkout <base> && git pull` |

## Edge Cases

**Worktree path has spaces**: always quote `"$CURRENT_WORKTREE_PATH"` in shell commands.

**PR already exists for this branch**: `gh pr create` will error. Catch it:
```bash
gh pr create ... 2>&1 | grep -q "already exists" && gh pr view --web
```
In this case, just print the existing PR URL and proceed to cleanup.

**Detached HEAD**: if `git branch --show-current` returns empty, stop and tell the user: "Cannot ship from detached HEAD state. Check out a named branch first."

**No remote configured**: if `git remote` returns nothing, stop and tell the user to configure a remote first.

## Red Flags

**Never:**
- Ship with failing tests
- Force-push without the user explicitly asking
- Delete uncommitted changes without confirmation
- Assume the base branch name without detecting it

**Always:**
- Detect worktree vs. regular branch before acting
- Pull latest on base branch after switching (regular branch case)
- Print the PR URL clearly after creation
- Give explicit `cd` instructions when removing a worktree — you cannot change the user's shell CWD
- Remind the user if a stash was created during Step 2
