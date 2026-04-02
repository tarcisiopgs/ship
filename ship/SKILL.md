---
name: ship
description: Use this skill when the user wants to ship work via a Pull Request and automatically return the repository to its original clean state. Trigger when the user says "ship", "ship it", "ship this PR", "create a PR and cleanup", "done with this branch", "push and go back", "finalize this feature", or any variation of wanting to open a PR and restore the workspace — whether they're on a regular branch, inside a git worktree, or working across multiple independent repos in a monorepo-style parent directory. Always use this skill instead of manually creating PRs when cleanup is expected afterward.
---

# Ship

Ship the current work via a Pull Request and restore the repository to its original state.

**Announce at start:** "I'm using the ship skill to create your PR and clean up the workspace."

## Overview

This skill is opinionated and streamlined: it always ships via PR and always restores the repository afterward. No menu, no options — just push, PR, and return to clean state.

The full flow:
1. Detect context (worktree, single repo, or multi-repo parent)
2. [Multi-repo] Discover all affected sibling repos and iterate Steps 3–7 for each
3. Handle uncommitted changes
4. Verify tests pass
5. Push and create PR (with assignee auto-set to current user)
6. Add reviewers (optional, excluding the current user)
7. Monitor CI — fix failures automatically when possible, escalate when not
8. Restore original state
9. [Multi-repo] Summarize all PRs

---

## Step 1: Detect Context

Run these commands and capture the output:

```bash
git worktree list 2>/dev/null
git rev-parse --show-toplevel 2>/dev/null
git branch --show-current 2>/dev/null
```

**Three possible contexts:**

### A) Worktree context
Current path matches a non-first line in `git worktree list`. Proceed with single-repo flow (Steps 3–8) inside this worktree.

### B) Regular single-repo branch
Current path is the main worktree path (first line of `worktree list`). Proceed with single-repo flow (Steps 3–8).

### C) Multi-repo parent (no git root here, but subdirs are independent repos)
`git rev-parse --show-toplevel` fails or returns the CWD itself with no commits. Check for sibling repos:

```bash
for dir in */; do
  if [ -d "$dir/.git" ]; then
    echo "$dir"
  fi
done
```

If multiple subdirectories are independent git repos, **this is a multi-repo scenario** — go to Step 2.

Capture and store for single-repo flow:
- `CURRENT_BRANCH`: output of `git branch --show-current`
- `MAIN_WORKTREE_PATH`: first path from `git worktree list`
- `MAIN_WORKTREE_BRANCH`: branch from `git worktree list | head -1` (the part inside `[...]`)
- `CURRENT_WORKTREE_PATH`: output of `git rev-parse --show-toplevel` (only if worktree)
- `IS_WORKTREE`: boolean

Also resolve the current GitHub user now — you'll need it in Steps 5 and 6:

```bash
GITHUB_USER=$(gh api user --jq '.login' 2>/dev/null)
```

---

## Step 2: Multi-Repo Discovery (only if context is C)

When invoked from a parent directory containing multiple independent git repos, the user likely has work spread across more than one of them. You need to figure out which repos are actually affected.

**Discover affected repos:**

```bash
for dir in */; do
  [ -d "$dir/.git" ] || continue
  cd "$dir"
  branch=$(git branch --show-current 2>/dev/null)
  uncommitted=$(git status --short 2>/dev/null | wc -l | tr -d ' ')
  ahead=$(git rev-list --count @{u}..HEAD 2>/dev/null || echo "0")
  if [ "$uncommitted" -gt 0 ] || [ "$ahead" -gt 0 ]; then
    echo "$dir (branch: $branch, uncommitted: $uncommitted, ahead: $ahead)"
  fi
  cd ..
done
```

A repo is "affected" if it has uncommitted changes **or** commits not yet pushed to origin.

**Present findings to the user:**

```
I found changes in the following repos:

  • api/         (branch: feat/DEV-1040, 3 commits ahead, 0 uncommitted)
  • corporate/   (branch: feat/DEV-1040, 5 commits ahead, 0 uncommitted)

I'll create a PR for each one. Proceed?
```

Wait for confirmation, then **iterate Steps 3–8 for each affected repo in sequence**, `cd`-ing into each one before starting. Keep a running list of all PR URLs created.

After all repos are done, go to Step 9.

---

## Step 3: Handle Uncommitted Changes

*(Run this per-repo when in multi-repo mode — `cd` into the repo first)*

```bash
git status --short
```

If there are uncommitted changes, ask:

```
There are uncommitted changes in <repo>. What would you like to do?

1. Commit them now (I'll write a commit message based on the changes)
2. Stash them
3. Cancel — I'll handle this myself

Which option?
```

- **Option 1**: Stage all and commit with a message derived from `git diff` summary
- **Option 2**: `git stash push -m "ship: stashed before PR"` — remind the user to pop stash later
- **Option 3**: Stop entirely

If there are no uncommitted changes, continue silently.

---

## Step 4: Verify Tests

*(Run this per-repo — auto-detect the test suite from within the repo directory)*

```bash
# Priority order — use the first match
if [ -f "yarn.lock" ] && grep -q '"test"' package.json 2>/dev/null; then
  yarn test --run
elif [ -f "bun.lockb" ] || grep -q '"test"' package.json 2>/dev/null; then
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
Tests failing in <repo> — cannot create PR with broken tests.

[paste failures]

Fix the failures and run /ship again.
```
Stop.

**If no test suite is detected:** Warn the user and ask whether to proceed without test verification.

**If tests pass:** Continue silently.

---

## Step 5: Detect Origin Branch, Push, and Create PR

*(Run this per-repo)*

The origin branch is **where the current branch was created from** — not necessarily `main`. This is what the PR will target and where the workspace will return to.

**Detection strategy (run in order, use first result that makes sense):**

For worktrees — the main worktree's branch is the strongest signal:
```bash
ORIGIN_BRANCH=$(git worktree list | head -1 | grep -oP '\[\K[^\]]+')
```

For regular branches — find the local branch with the fewest unique commits ahead:
```bash
CANDIDATES=$(git for-each-ref --format='%(refname:short)' refs/heads/ | grep -v "^$(git branch --show-current)$")
BEST=""
BEST_AHEAD=999999
for candidate in $CANDIDATES; do
  ahead=$(git rev-list --count "$candidate"..HEAD 2>/dev/null || echo 999999)
  if [ "$ahead" -lt "$BEST_AHEAD" ]; then
    BEST_AHEAD=$ahead
    BEST=$candidate
  fi
done
ORIGIN_BRANCH=$BEST
```

Fallback to remote default:
```bash
if [ -z "$ORIGIN_BRANCH" ]; then
  ORIGIN_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
fi
if [ -z "$ORIGIN_BRANCH" ]; then
  ORIGIN_BRANCH="main"
fi
```

**Confirm with the user** if uncertain (multiple candidates with the same distance, or fallback was used):
```
Detected origin branch for <repo>: <ORIGIN_BRANCH> (<N> commits behind HEAD).
Is this correct?
```

**Push:**
```bash
git push -u origin "$CURRENT_BRANCH"
```

**Create the PR — always assign to the current user.**

Setting the current user as assignee is important because it makes ownership clear in GitHub: who shipped this feature. You already resolved `GITHUB_USER` in Step 1.

Build the PR title from the branch name: strip common prefixes (`feat/`, `fix/`, `feature/`, `chore/`, `refactor/`, `hotfix/`), replace hyphens with spaces, capitalize. Cross-reference with recent commits to make it human-readable.

Build the PR body from `git log $ORIGIN_BRANCH..HEAD` and `git diff $ORIGIN_BRANCH...HEAD --stat`:

```bash
PR_URL=$(gh pr create \
  --title "<derived title>" \
  --base "$ORIGIN_BRANCH" \
  --assignee "$GITHUB_USER" \
  --body "$(cat <<'EOF'
## Summary
<2-3 bullets summarizing what changed, derived from commits and diff>

## Test Plan
- [ ] <key verification steps based on what changed>

🤖 Shipped via [ship skill](https://github.com/tarcisiopgs/ship-skill)
EOF
)")
```

Record the PR URL. In multi-repo mode, add it to the running list and move on to the next repo.

---

## Step 6: Add Reviewers (optional)

After the PR is created, ask the user if they want to add reviewers:

```
PR created: <URL>

Would you like to add reviewers? (yes/no)
```

**If yes:**

Fetch the repository's contributors, excluding yourself (since you're already the assignee, adding yourself as reviewer is redundant and GitHub won't allow it anyway):

```bash
gh api repos/{owner}/{repo}/contributors --jq '.[].login' 2>/dev/null | grep -v "^$GITHUB_USER$" | head -20
```

Extract `{owner}` and `{repo}` from the remote URL:
```bash
git remote get-url origin | sed 's/.*github.com[:/]\(.*\)\.git/\1/' | awk -F'/' '{print $1, $2}'
```

Present the contributors as a numbered list:

```
Available contributors:

  1. alice
  2. bob
  3. carol

Who should review this PR? (enter names or numbers, comma-separated)
```

Wait for the user's answer, then add the reviewers:

```bash
gh pr edit "<PR_URL>" --add-reviewer "alice,bob"
```

Confirm: "Reviewers added: alice, bob"

**If no (or no answer within context):** Skip silently and proceed to Step 7.

In multi-repo mode: ask about reviewers per repo, immediately after each PR is created.

---

## Step 7: Monitor CI

After the PR is created, check whether CI exists for this repository. CI runs give you a feedback loop that closes the loop on "did this actually work?" — don't skip monitoring if CI is present.

**Check for CI:**
```bash
gh pr checks "$PR_URL" 2>/dev/null
```

If the output is empty or returns "no checks", there is no CI configured — skip this step and continue to Step 8.

If CI checks exist, enter the monitoring loop:

### Monitoring loop

```bash
# Poll until all checks finish (pass or fail)
gh pr checks "$PR_URL" --watch 2>/dev/null
```

**If all checks pass:** Report success and proceed to Step 8.

**If checks fail:** Read the failure details:

```bash
gh run list --branch "$CURRENT_BRANCH" --limit 5 --json databaseId,name,status,conclusion 2>/dev/null
# For each failed run, get the logs:
gh run view <run_id> --log-failed 2>/dev/null
```

Then decide: **can this be fixed automatically?**

A failure is fixable if it's one of these:
- Lint or formatting errors (run the linter/formatter, commit, push)
- Type errors introduced by the changes (fix the types, commit, push)
- Test failures caused by a mock not being updated for a new hook or export (update the mock, commit, push)
- Missing i18n keys referenced in the code (add the key, commit, push)

A failure is **not fixable** without the user if it involves:
- Flaky infrastructure (network timeouts, third-party services)
- Secrets or environment variables missing from CI
- Failures in tests completely unrelated to the current changes
- Logic errors that require product judgment to resolve
- Any failure where you've already attempted a fix and it failed again

**When fixing:**
1. Apply the fix
2. Commit with a clear message (e.g., `fix: resolve CI lint errors`)
3. Push to the same branch
4. Re-enter the monitoring loop — wait for the new run to complete

**Loop limit:** After 3 fix attempts, stop even if the cause seems fixable. Repeating the same fix approach without progress is a signal that something deeper is wrong.

**When escalating to the user:**

```
CI is failing and I wasn't able to fix it automatically.

Failing checks:
  • <check name> — <brief description of error>

Relevant logs:
<paste the most relevant failure output, trimmed to what matters>

What would you like to do?
```

Wait for the user's instructions before proceeding.

---

## Step 8: Restore Original State

*(Run this per-repo immediately after CI passes or the user resolves the failure)*

### If in a Worktree

```bash
git worktree remove "$CURRENT_WORKTREE_PATH"
```

Since you cannot change the user's shell working directory, **always print explicit instructions**:

```
Worktree removed. Return to your origin branch:

  cd <MAIN_WORKTREE_PATH>
  git checkout <ORIGIN_BRANCH>   # (only if main worktree isn't already on it)
  git pull
```

If `MAIN_WORKTREE_BRANCH` already equals `ORIGIN_BRANCH`, omit the `git checkout` line.

### If on a Regular Branch

```bash
git checkout "$ORIGIN_BRANCH"
git pull
```

In multi-repo mode: stay in the repo directory long enough to restore, then `cd ..` back to the parent before starting the next repo.

---

## Step 9: Multi-Repo Summary (only if Step 2 ran)

After all repos are processed, print a clean summary:

```
All PRs created:

  • api/       → https://github.com/org/api/pull/616
  • corporate/ → https://github.com/org/corporate/pull/284

Branches restored to <ORIGIN_BRANCH> in all repos.
```

If a stash was created in any repo during Step 3, remind the user:
```
Don't forget to pop your stash in <repo>:
  cd <repo> && git stash pop
```

---

## Quick Reference

| Context | Detection | Flow |
|---------|-----------|------|
| Worktree | `git worktree list` shows current path as non-first | Steps 3–8 once |
| Regular branch | Single `.git` at CWD | Steps 3–8 once |
| Multi-repo parent | `git rev-parse --show-toplevel` fails; subdirs have `.git` | Step 2 → Steps 3–8 per repo → Step 9 |

---

## Edge Cases

**Worktree path has spaces**: always quote `"$CURRENT_WORKTREE_PATH"` in shell commands.

**PR already exists for this branch**: `gh pr create` will error. Catch it:
```bash
PR_OUTPUT=$(gh pr create ... 2>&1)
if echo "$PR_OUTPUT" | grep -q "already exists"; then
  PR_URL=$(gh pr view --json url --jq '.url')
else
  PR_URL="$PR_OUTPUT"
fi
```
Record the existing PR URL and continue.

**Repos with different branch names**: In multi-repo mode, each repo may be on a different branch. Report the branch name per repo in the discovery step and handle each independently.

**Detached HEAD**: if `git branch --show-current` returns empty, skip that repo with a warning: "Skipping <repo> — detached HEAD state."

**No remote configured**: if `git remote` returns nothing for a repo, warn and skip it.

**One repo fails tests**: In multi-repo mode, stop entirely and report which repo failed. Don't ship any repo if one has broken tests — the change is probably cross-repo and partial shipping creates inconsistency.

**CI flapping**: if a check fails on the first run but passes on the second without any code change, it was flaky infrastructure. Note it to the user but don't treat it as a blocker.

---

## Red Flags

- Don't ship with failing tests (in any repo)
- Don't force-push without the user explicitly asking
- Don't delete uncommitted changes without confirmation
- Don't assume the origin branch is always `main` — detect it properly
- Don't target the wrong base on `gh pr create` (always pass `--base "$ORIGIN_BRANCH"`)
- Don't ship only some repos in a multi-repo scenario without warning the user
- Don't keep attempting the same CI fix if it didn't work the first time — escalate instead
- Detect worktree vs. regular branch vs. multi-repo parent before acting
- Pull latest on origin branch after switching, in every repo
- Print each PR URL clearly after creation
- Give explicit `cd` + `git checkout` instructions when removing a worktree — you cannot change the user's shell CWD
- Remind the user if a stash was created during Step 3
