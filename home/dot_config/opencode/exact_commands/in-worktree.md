---
description: Implement a feature in an isolated git worktree at /tmp
---

Implement the following feature in a fresh git worktree, keeping the current repo untouched:

**Feature:** $ARGUMENTS

## Setup

### 1. Derive names

From the feature description, produce:

- **branch-slug**: lowercase kebab-case, max 5 words, no special chars (e.g. `add-oauth-login-flow`)
- **worktree-path**: `/tmp/<repo-name>-<branch-slug>`; `<repo-name>` is the basename of the current repo root

### 2. Fetch and resolve base branch

```bash
git fetch origin
```

Then resolve the default remote branch. `git fetch origin` must run first so `refs/remotes/origin/HEAD` exists.

```bash
ref=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null) || true
base_branch=${ref#origin/}
```

If `base_branch` is empty, run `git remote set-head origin -a` once, then repeat the assignment. Abort and report if `base_branch` is still empty.

### 3. Create the worktree

Remove any stale worktree at the target path first, then create a new branch and worktree:

```bash
git worktree remove --force <worktree-path> 2>/dev/null || true
git worktree add -b <branch-slug> <worktree-path> origin/<base-branch>
```

### 4. Verify working directory

Run inside the worktree to confirm you are on the right branch and have a clean slate:

```bash
git -C <worktree-path> status
```

The output must show the new branch name and a clean working tree. If it does not, stop and report the discrepancy before doing any work.

### 5. Toolchain in the worktree (non-interactive)

Bootstrap tools using only `<worktree-path>` so tests, linters, and task runners work without prompts. That matches default OpenCode `permission.bash` rules: no extra approval for `mise trust` or installs under the worktree path.

- **mise** (if `.mise.toml`, `mise.toml`, or `.tool-versions` exists under `<worktree-path>`): run `mise -C <worktree-path> trust`, then `mise -C <worktree-path> install -y` when tools must be present to run tests or linters.
- **Other manifests** (e.g. `package.json`, `go.mod`, `requirements.txt`, `*.csproj`): run the project’s usual install with cwd `<worktree-path>`.

Skip steps that do not apply. Do not run `sudo` or destructive system commands unless the user explicitly asked for them.

## Implementation

**All file reads, writes, and shell commands must use the worktree path as the working directory.** Never modify files in the original repo root.

Follow the standard implementation loop:

1. Explore the worktree to understand the codebase structure relevant to the feature
2. Plan the changes needed
3. Implement, running tests and linters from within the worktree as appropriate
4. Verify the final state with `git -C <worktree-path> diff origin/<base-branch>` so only the intended changes are present

## Rules

- Use absolute paths rooted at `<worktree-path>` for every file operation
- Do not `cd` into the worktree in a way that could silently fall back to the original repo. Always pass `-C <worktree-path>` to git commands or use absolute paths.
- Do not push without explicit user approval
- Leave the worktree in place when done so the user can review, test, and push
- Report the worktree path and branch name at the end so the user knows where to find the work
