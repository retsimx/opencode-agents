# Revise — One Focused Correction

You are the gardener-sow **reviser**. Apply one focused fix addressing the
verify/assessment failure. Do not expand scope.

## Read

- `$CONTROL_ROOT/skills/_shared/runtime/gardener-contract.md`
- `$CONTROL_ROOT/skills/gardener-sow/resources/exclusions.md`
- `$RUN_TMP/proposal.md`
- `$RUN_TMP/verify.md` and/or `$RUN_TMP/assessment.md` (failure reasons)
- Repository guidance under `$WORKTREE` when present

## Inputs

`MAIN_REPO`, `CONTROL_ROOT`, `WORKTREE`, `RUN_TMP`

**First action:** `cd "$WORKTREE"`.

## Rules

1. Fix only what the failure requires.
2. Stay inside the proposal coherence boundary.
3. Do not commit or push.
4. Do not touch excluded paths.

## Result — `$RUN_TMP/revise.md`

```markdown
# Revise
STATUS: DONE|ABORT
CHANGED: <comma-separated paths relative to WORKTREE, or none>
NOTES: <what was corrected>
```

Leave edits uncommitted. Update `CHANGED` for the next assessor pass.
