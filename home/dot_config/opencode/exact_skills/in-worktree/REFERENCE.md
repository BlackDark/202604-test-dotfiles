# in-worktree: toolchain (optional)

Read only when you need to bootstrap tools inside the worktree. Assume `ACTIVE_WORKDIR` is already
set and verified per SKILL.md.

Use only `"$ACTIVE_WORKDIR"` so installs stay under that tree and stay non-interactive where
possible. Skip steps that do not apply. No `sudo` or destructive system commands unless the user
explicitly asked.

## mise

If `.mise.toml`, `mise.toml`, or `.tool-versions` exists under `"$ACTIVE_WORKDIR"`:

```bash
mise -C "$ACTIVE_WORKDIR" trust
mise -C "$ACTIVE_WORKDIR" install -y
```

Run the second command when tools must be present for tests or linters.

## Other manifests

Examples: `package.json`, `go.mod`, `requirements.txt`, `*.csproj`. Run the project usual install
with cwd in the worktree:

```bash
cd "$ACTIVE_WORKDIR" && <usual install command>
```

## rg and search

From SKILL / AGENTS: prefer `rg`. When searching the feature tree only, scope under the worktree:

```bash
cd "$ACTIVE_WORKDIR" && rg "pattern"
```

Or pass path arguments that all lie under `"$ACTIVE_WORKDIR"`.

## gh

For repo-scoped GitHub CLI work tied to this checkout, prefer:

```bash
cd "$ACTIVE_WORKDIR" && gh <subcommand> ...
```

Or set `-R owner/repo` from `git -C "$ACTIVE_WORKDIR" remote -v`. Do not assume the CLI infers the
correct repo from the editor workspace cwd.
