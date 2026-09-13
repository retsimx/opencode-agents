# Gardener Contract

Normative rules for `gardener-sow`, `gardener-tend`, and `gardener-harvest`.
Load this file before any gardener work.

## Micro-improvement

A gardener PR:

1. Addresses one coherent concern.
2. Has concrete evidence that the concern exists.
3. Is worth a permanent commit, CI, and forge record.
4. Preserves or improves intended correctness.
5. Adds no more complexity than the improvement requires.
6. Uses existing project mechanisms where they fit.
7. Includes verification proportional to the risk.

Bug fixes, useful new tests, dead-code removal, and small refactors are valid.
Behavior change is valid only with evidence from a spec, tests, callers,
documented bug, framework contract, or consistent project pattern. Agent
preference is not evidence.

Cosmetic motion is not automatically improvement. Work that needs a coordinated
architecture change is out of scope — report it, do not fragment it.

## Identification

A gardener PR has all of:

- title prefix `chore(gardener):`
- branch prefix `gardener/run-`
- target equal to the resolved default branch
- source branch on the target repository (not a fork)

Do not recognize `gardener/iter-` or other prior names.

## Comment marker

Every gardener-authored comment must include this exact line:

`<!-- gardener -->`

`rejected` / suppress is allowed only when a **non-marked** comment explicitly
asks to reject or close. Agent CLOSE comments are marked and must not suppress.

## Grug

All Grug principles apply. Discuss only principles that matter for this change.
Do not print a full rule table.

## Concurrency

The outer loop may run only one gardener skill at a time against the same
`CONTROL_ROOT` + `MAIN_REPO`. That is an outer-loop rule, not an in-skill lock.
`intent.json` is crash recovery, not a mutex.

## Paths

Resolve once per invocation:

- `MAIN_REPO` — target git repository (forge + worktree add).
- `CONTROL_ROOT` — walk parents of this skill file until a directory contains
  `skills/gardener-sow`. State and temp files live here. It may sit inside
  `MAIN_REPO`.
- `WORKTREE` — `$CONTROL_ROOT/worktrees/gardener-<run-id>` (sow / tend edits).
- `RUN_DIR` — `$CONTROL_ROOT/results/tmp/pr-harvest/<run-id>/` (harvest assessor).

Registry, intent, and temp writes under `CONTROL_ROOT` are allowed even when
`CONTROL_ROOT` is inside `MAIN_REPO`. Do not edit tracked **project** source in
the main checkout. Source edits and tests run in `WORKTREE`.

## Local checks

In this order, and never invent a runner or install tooling:

1. A command named in repository guidance, if locally executable without extra
   secrets or installs.
2. A script named by that repository's CI config, if locally executable without
   extra secrets or installs.
3. Otherwise no local suite — use forge checks and `gardener-tend`.

## Human override

Any commenter may request a change. The PR must stay one coherent improvement
and must still obey project hard rules and system safety.
