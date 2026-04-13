---
description: Implement a feature in an isolated git worktree at /tmp
---

Implement the following feature in a fresh git worktree, keeping the current repo untouched:

**Feature:** $ARGUMENTS

## Required

Load the `in-worktree` skill alone before any file read, write, or shell command. If you cannot load
it, stop and say so.

## Directory discipline

As soon as `ACTIVE_WORKDIR` is known (see skill), send **one** user-visible line exactly in this
form (literal absolute path, no env var reference in the line):

`Locked workdir: /full/path/to/worktree`

**Every** file tool call (`read`, `write`, `edit`, and similar) MUST use paths under
`ACTIVE_WORKDIR` after `realpath` when you compare prefixes (handles symlinks and `/private/var` on
macOS). Do not use workspace-relative paths for edits during this task.

**Git:** only `git -C "$ORIGINAL_ROOT"` during bootstrap, then only `git -C "$ACTIVE_WORKDIR"`;
no bare `git`. **gh:** `cd "$ACTIVE_WORKDIR" && gh ...` or explicit `-R` from that repo remote.

Re-run the skill verification before `git add`, `git commit`, `git push`, and before each batch of
file edits. Before commit, run the mechanical `status -sb` and `diff --name-only` checks from the
skill.
