---
name: in-worktree
description: >-
  Use when running the /in-worktree slash command, implementing a feature in a git worktree under
  /tmp, or whenever WORKTREE_ROOT is established for an isolated clone of the current repository
---

# Isolated git worktree workflow

Purpose: implement work only inside a dedicated worktree. The original repository checkout must
never receive edits, commits, or pushes from this flow.

## 1. Lock the original repo root (first shell commands)

Run this before any other shell command, file tool, or subagent in this flow:

```bash
ORIGINAL_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd -P)
```

Use `"$ORIGINAL_ROOT"` in path form for file operations only when reading the pre-worktree tree is
required. All implementation work happens under `"$WORKTREE_ROOT"` after it exists.

## 2. Names and base branch

From the feature description, produce:

- **branch-slug**: lowercase kebab-case, max 5 words, no special chars (e.g. `add-oauth-login-flow`)
- **worktree-path**: `/tmp/<repo-name>-<branch-slug>`; `<repo-name>` is the basename of
  `"$ORIGINAL_ROOT"`

Resolve `base_branch` using the main repo only:

```bash
git -C "$ORIGINAL_ROOT" fetch origin
ref=$(git -C "$ORIGINAL_ROOT" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null) || true
base_branch=${ref#origin/}
```

If `base_branch` is empty, run `git -C "$ORIGINAL_ROOT" remote set-head origin -a` once, then repeat
the assignment. Abort if still empty.

## 3. Create the worktree

```bash
git -C "$ORIGINAL_ROOT" worktree remove --force "<worktree-path>" 2>/dev/null || true
git -C "$ORIGINAL_ROOT" worktree add -b "<branch-slug>" "<worktree-path>" "origin/${base_branch}"
```

## 4. Canonical worktree path and verification

Assign the path you will use for every later command:

```bash
WORKTREE_ROOT=$(cd "<worktree-path>" && pwd -P)
```

Fail fast if any check fails (stop and report; do not edit files):

```bash
test -n "$WORKTREE_ROOT"
test -n "$ORIGINAL_ROOT"
test "$WORKTREE_ROOT" != "$ORIGINAL_ROOT"
test "$(git -C "$WORKTREE_ROOT" rev-parse --show-toplevel)" = "$WORKTREE_ROOT"
```

Re-run this four-line check after subshells, subagent handoffs, and **immediately before** `git add`,
`git commit`, `git push`, or any batch of file writes. If a check fails, fix paths before continuing.

Confirm state:

```bash
git -C "$WORKTREE_ROOT" status
```

Output must show the new branch and a clean tree before implementation.

## 5. Working directory rules

- **Git**: use `git -C "$WORKTREE_ROOT" ...` for every git subcommand on the feature branch. Do not
  run bare `git` when the shell cwd might still be `"$ORIGINAL_ROOT"` or elsewhere.
- **Non-git tools** (package managers, tests, linters, builds): one line per invocation so cwd
  cannot drift, e.g. `cd "$WORKTREE_ROOT" && <command>`.
- **File reads and writes**: paths under `"$WORKTREE_ROOT"` only. If a tool takes a repo root, pass
  `"$WORKTREE_ROOT"`.

## 6. Toolchain (non-interactive)

Bootstrap tools using only `"$WORKTREE_ROOT"`. For mise, if a config exists under the worktree: `mise
-C "$WORKTREE_ROOT" trust` then `mise -C "$WORKTREE_ROOT" install -y` when needed. For other
manifests, `cd "$WORKTREE_ROOT" &&` with the project usual install.

Skip steps that do not apply. No `sudo` or destructive system commands unless the user explicitly
asked.

## 7. Implementation loop

1. Explore and plan under `"$WORKTREE_ROOT"`.
2. Implement; run tests and linters with `cd "$WORKTREE_ROOT" && ...`.
3. Review diff: `git -C "$WORKTREE_ROOT" diff "origin/${base_branch}"` (adjust if comparing to
   another base).

## 8. Before commit

1. Run the verification block from section 4.
2. `git -C "$WORKTREE_ROOT" add ...` and `git -C "$WORKTREE_ROOT" commit ...` only; never commit from
   `"$ORIGINAL_ROOT"` in this flow.

## 9. Before push

1. Run the verification block from section 4 again.
2. Do **not** push unless the user explicitly approved pushing in this conversation.
3. If approved: `git -C "$WORKTREE_ROOT" push -u origin "<branch-slug>"` (adjust only if the user
   asked).

## 10. Subagents and delegation

Pass `"$WORKTREE_ROOT"` and `"$ORIGINAL_ROOT"` in text. Subagents must use `git -C "$WORKTREE_ROOT"`
and paths under `"$WORKTREE_ROOT"` only; they must not assume workspace cwd is the worktree.

## 11. Completion

Leave the worktree on disk. Report `"$WORKTREE_ROOT"` and the branch name.
