# ship

A Claude Code skill that ships your current work via a Pull Request and automatically restores the repository to its original state — whether you're on a regular branch or inside a git worktree.

## What it does

1. Detects context (git worktree or regular branch)
2. Handles uncommitted changes (commit, stash, or cancel)
3. Verifies tests pass before shipping
4. Pushes the branch and creates a PR via `gh`
5. Restores the workspace:
   - **Worktree**: removes the worktree and tells you to `cd` back to main
   - **Regular branch**: checks out the base branch and pulls latest

## Install

```bash
npx skills add tarcisiopgs/ship@ship -g
```

## Usage

Just say **"ship"** or **"ship it"** in Claude Code. The skill handles the rest.

## Requirements

- [`gh`](https://cli.github.com/) (GitHub CLI) — for creating PRs
- Git remote configured and authenticated
