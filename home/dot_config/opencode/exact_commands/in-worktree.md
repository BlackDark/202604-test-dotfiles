---
description: Implement a feature in an isolated git worktree at /tmp
---

Implement the following feature in a fresh git worktree, keeping the current repo untouched:

**Feature:** $ARGUMENTS

## Setup

### 1. Derive names

From the feature description, produce:

- **branch-slug**: lowercase kebab-case, max 5 words, no special chars (e.g.,
  `add-oauth-login-flow`)
- **worktree-path**: `/tmp/<repo-name>-<branch-slug>` where `<repo-name>` is the basename of the
  current repo root

### 2. Fetch and resolve base branch

```bash
git fetch origin
```

Then detect the default remote branch:

```bash
git ls-remote --symref origin HEAD | awk '/^ref:/ {print $2}' | sed 's|refs/heads/||'
```

Use `main` if the result is `main`, `master` if the result is `master`, otherwise use whatever the
command returns. Abort and report if neither is found.

### 3. Create the worktree

Remove any stale worktree at the target path first, then create a new branch and worktree in one
command:

```bash
git worktree remove --force <worktree-path> 2>/dev/null || true
git worktree add -b <branch-slug> <worktree-path> origin/<base-branch>
```

### 4. Verify working directory

Run inside the worktree to confirm you are on the right branch and have a clean slate:

```bash
git -C <worktree-path> status
```

The output must show the new branch name and a clean working tree. If it does not, stop and report
the discrepancy before doing any work.

## Implementation

**All file reads, writes, and shell commands must use the worktree path as the working directory.**
Never modify files in the original repo root.

Follow the standard implementation loop:

1. Explore the worktree to understand the codebase structure relevant to the feature
2. Plan the changes needed
3. Implement, running tests and linters from within the worktree as appropriate
4. Verify the final state with `git -C <worktree-path> diff origin/<base-branch>` to confirm only
   the intended changes are present

## Rules

- Use absolute paths rooted at `<worktree-path>` for every file operation
- Do not `cd` into the worktree in a way that could silently fall back to the original repo; always
  pass `-C <worktree-path>` to git commands or use absolute paths
- Do not push without explicit user approval
- Leave the worktree in place when done so the user can review, test, and push
- Report the worktree path and branch name at the end so the user knows where to find the work
