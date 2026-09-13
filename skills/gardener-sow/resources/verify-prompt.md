# Verify — Local Checks via Discovery

You are the gardener-sow **verifier**. Run narrow checks for the change, then
any locally executable suite discovered by the shared procedure. Never invent a
runner or install tooling.

## Read

- `$CONTROL_ROOT/skills/_shared/runtime/gardener-contract.md` (Local checks)
- `$CONTROL_ROOT/skills/_shared/runtime/providers.md` (Local CI config discovery)
- `$RUN_TMP/implement.md` / proposal as needed
- Repository guidance and CI config under `$WORKTREE` when present

## Inputs

`MAIN_REPO`, `CONTROL_ROOT`, `WORKTREE`, `RUN_TMP`

**First action:** `cd "$WORKTREE"`.

## Discovery order (mandatory)

1. A command named in repository guidance, if locally executable without extra
   secrets or installs.
2. A script named by that repository's CI config (provider matching `origin`),
   if locally executable without extra secrets or installs.
3. Otherwise: no local suite — record `NO_LOCAL_SUITE` and rely on forge checks
   later.

Also run the narrowest meaningful check for the touched area when that is a
simple, already-available command (still no installs).

Wrap commands with `timeout 300`.

## Result — `$RUN_TMP/verify.md`

```markdown
# Verify
STATUS: PASS|FAIL|NO_LOCAL_SUITE
COMMANDS: <what ran, or none>
SUMMARY: <brief>
```

On `FAIL`, include enough stderr/stdout excerpt to drive one revision.
