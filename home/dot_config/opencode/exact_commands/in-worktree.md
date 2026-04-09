---
description: Implement a feature in an isolated git worktree at /tmp
---

Implement the following feature in a fresh git worktree, keeping the current repo untouched:

**Feature:** $ARGUMENTS

## Required

You MUST load the `in-worktree` skill alone, before any file read, write, or shell command, and follow
it completely. The skill defines `ORIGINAL_ROOT`, `WORKTREE_ROOT`, verification gates, and how to run
git and non-git tools.

If you cannot load the skill, stop and say so; do not improvise this workflow.

## Reminder

Re-run the skill verification block immediately before `git add`, `git commit`, or `git push`, and
before any batch of file edits, so you never operate in `"$ORIGINAL_ROOT"` by mistake.
