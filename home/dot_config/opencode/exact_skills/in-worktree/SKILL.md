---
name: in-worktree
description: >-
  Use when the user runs /in-worktree, or when only a /tmp worktree may receive edits for the task
---

# In-worktree session

## Memory card

The editor workspace root is **not** the workdir. Relative paths in file tools resolve there and
will edit the **wrong** tree.

After setup below yields a real directory, **pin** it for the entire task:

```txt
ACTIVE_WORKDIR=<absolute path from pwd -P only, never a placeholder>
```

Until the task ends:

- **Forbidden**: any read, write, or edit on a path that is not under `ACTIVE_WORKDIR` after
  `realpath` (when `realpath` exists). If paths differ only by symlinks (`/var` vs `/private/var`),
  normalize before comparing.
- **Git**: **no** `git` without `-C` in this workflow. Use `git -C "$ORIGINAL_ROOT"` only until
  `ACTIVE_WORKDIR` is set; after that, use only `git -C "$ACTIVE_WORKDIR"` for all feature git.
- **Shell** (tests, installs, linters): `cd "$ACTIVE_WORKDIR" && ...` as one line per invocation.
- **`gh`**: run repo-scoped commands as `cd "$ACTIVE_WORKDIR" && gh ...`, or pass an explicit
  `-R owner/repo` taken from `git -C "$ACTIVE_WORKDIR" remote -v`. Do not rely on cwd for `gh`.
- **Other mutators** (shell redirects, MCP or other tools that write files): destinations must
  resolve under `ACTIVE_WORKDIR` (same `realpath` rule).

Re-pin after subagent handoffs. Re-run the verification block before any batch of file edits or any
`git add`, `commit`, or `push`.

Optional: if the user can open `ACTIVE_WORKDIR` as the editor workspace, that avoids path mistakes.

Optional detail: [REFERENCE.md](REFERENCE.md) (toolchain bootstrap).

## Setup

First shell command in this flow. Start from inside the **main** checkout you intend to branch from:

```bash
ORIGINAL_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd -P)
```

From the feature description, choose **BRANCH_SLUG**: lowercase kebab-case, max 5 words, no special
chars (example: `add-oauth-login-flow`). Then set paths in shell (no angle-bracket placeholders):

```bash
BRANCH_SLUG="your-branch-slug-here"
WORKTREE_PATH="/tmp/$(basename "$ORIGINAL_ROOT")-${BRANCH_SLUG}"
```

**CRITICAL:** Replace `your-branch-slug-here` with a real slug before running anything that follows.
Never run git or file tools with a `BRANCH_SLUG` that still contains `your-branch-slug-here`.

Optional guard:

```bash
case "$BRANCH_SLUG" in *"<"*|*">"*|*"your-branch-slug-here"*)
  echo "abort: invalid BRANCH_SLUG"; exit 1;; esac
```

Base branch:

```bash
git -C "$ORIGINAL_ROOT" fetch origin
ref=$(git -C "$ORIGINAL_ROOT" symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null) || true
base_branch=${ref#origin/}
```

If `base_branch` is empty, run `git -C "$ORIGINAL_ROOT" remote set-head origin -a` once, then repeat
the `ref` / `base_branch` lines. Abort if still empty.

Create worktree:

```bash
git -C "$ORIGINAL_ROOT" worktree remove --force "$WORKTREE_PATH" 2>/dev/null || true
git -C "$ORIGINAL_ROOT" worktree add -b "$BRANCH_SLUG" "$WORKTREE_PATH" "origin/${base_branch}"
```

Pin and verify (stop and report if any check fails):

```bash
ACTIVE_WORKDIR=$(cd "$WORKTREE_PATH" && pwd -P)
test -n "$ACTIVE_WORKDIR"
test -n "$ORIGINAL_ROOT"
test "$ACTIVE_WORKDIR" != "$ORIGINAL_ROOT"
test "$(git -C "$ACTIVE_WORKDIR" rev-parse --show-toplevel)" = "$ACTIVE_WORKDIR"
```

Confirm:

```bash
git -C "$ACTIVE_WORKDIR" status
```

Output must show the new branch and a clean tree before other work.

Immediately tell the user once: `Locked workdir: ` plus the exact `ACTIVE_WORKDIR` string so the
path stays in the thread.

## Work loop

Explore and implement using paths under `"$ACTIVE_WORKDIR"`. Review:

```bash
git -C "$ACTIVE_WORKDIR" diff "origin/${base_branch}"
```

Adjust the base ref if the task requires it.

## Commit and push

Run the verification block again. Before staging, run a **mechanical** check and read the output:

```bash
git -C "$ACTIVE_WORKDIR" status -sb
git -C "$ACTIVE_WORKDIR" diff --name-only
```

Resolve anything unexpected before `git -C "$ACTIVE_WORKDIR" add` / `commit`. Prefer normalizing
path prefixes with `realpath` if the lists look wrong versus files you meant to touch.

Stage and commit only with `git -C "$ACTIVE_WORKDIR" ...`.

Do **not** push unless the user explicitly approved in this conversation. If approved:

```bash
git -C "$ACTIVE_WORKDIR" push -u origin "$BRANCH_SLUG"
```

Use the real branch name variable; adjust remote or options only if the user asked.

## Delegation

Pass `"$ACTIVE_WORKDIR"` and `"$ORIGINAL_ROOT"` in text. Subagents must follow the same path,
`git -C`, and `gh` rules; they must not assume workspace cwd is the worktree.

## Completion

Leave the worktree on disk. Report `"$ACTIVE_WORKDIR"` and the branch name.
