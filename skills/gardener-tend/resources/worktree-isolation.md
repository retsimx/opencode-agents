# Gardener Tend — Worktree Isolation (FATAL)

**Any modification of tracked project source in `MAIN_REPO` is a FATAL ERROR.**

Registry/intent writes under `CONTROL_ROOT` are allowed even when
`CONTROL_ROOT` lives inside `MAIN_REPO`.

## Definitions

| Name | Meaning |
|------|---------|
| `MAIN_REPO` | Primary git checkout (forge + `git worktree add` cwd) |
| `CONTROL_ROOT` | Agent pack root containing `skills/gardener-sow` |
| `WORKTREE` | `$CONTROL_ROOT/worktrees/gardener-<run-id>` |
| `run-id` | Unique id for this tend action (prefer PR's `gardener/run-…` suffix) |

## Rules

1. Every source-changing subagent receives absolute `MAIN_REPO`, `CONTROL_ROOT`,
   and `WORKTREE`, and must `cd "$WORKTREE"` before read/edit/test of project
   source.
2. Subagents must not edit, stage, commit, or run project checks in `MAIN_REPO`.
3. `git worktree add` from `MAIN_REPO` does **not** require a clean main
   checkout.
4. SETUP/CLEANUP may run `git fetch`, `git worktree add|remove|prune` from
   `MAIN_REPO` without modifying tracked project files.
5. Capture the remote head SHA before content work. Push only with
   `--force-with-lease` against that SHA.
6. All rebase/commit operations are non-interactive (`GIT_EDITOR=true`,
   `GIT_SEQUENCE_EDITOR=true`).

## Setup

From `MAIN_REPO`:

```bash
git fetch origin "<default-branch>" "<pr-branch>"
mkdir -p "$CONTROL_ROOT/worktrees"
git worktree add "$WORKTREE" "origin/<pr-branch>"
# or: git worktree add -b <pr-branch> "$WORKTREE" origin/<pr-branch>
```

Record `worktree_path` on the tend intent before mutating the worktree.

## Cleanup

After a completed action (or on abandon with no remote effect):

```bash
git worktree remove --force "$WORKTREE"   # when safe
git worktree prune
```

On skill **entry**, remove every `$CONTROL_ROOT/worktrees/gardener-*` path that
is **not** `intent.worktree_path`.

## Fatal error

If a subagent modifies files outside `WORKTREE` (except allowed `CONTROL_ROOT`
state/intent), or runs destructive git on `MAIN_REPO` tracked files:

```text
FATAL|main_modified|<detail>
```

Stop the invocation; report the path; do not push.

## Allowed in MAIN_REPO

- `git fetch`
- `git worktree add|remove|list|prune`
- Forge CLI reads/mutations that do not edit local project source
- Atomic writes to `$CONTROL_ROOT/state/gardener/*` when `CONTROL_ROOT ⊆ MAIN_REPO`

## Prohibited in MAIN_REPO

- Checkout/commit/reset/clean that changes tracked project files
- Editing project source files
- Running discovered local check suites against `MAIN_REPO` instead of `WORKTREE`
