# Gardener Tend — Scenario Fixtures

Lightweight expected behaviors for validation. Not executable tests.

## S1 — Human reject

- Setup: open gardener PR; non-marked comment "please close this"
- Expect: `tend_close`; PR closed; `closed.json` item `rejected` +
  `suppress_equivalent: true`; open entry removed; gardener rationale marked
- Must not: delete the human comment; treat a marked agent comment as rejection

## S2 — Rebase behind

- Setup: same-repo `gardener/run-*` PR behind default; no reject comments
- Expect: worktree under `$CONTROL_ROOT/worktrees/gardener-<run-id>`;
  non-interactive rebase; discovery checks; assessment PASS; force-with-lease;
  intent cleared
- Must not: push if remote head moved; edit `MAIN_REPO` source

## S3 — Rebase race

- Setup: captured head SHA H1; remote advances to H2 before push
- Expect: lease reject / abort; no overwrite; intent reconcile reports race

## S4 — Unrelated CI

- Setup: failed check clearly from base/infra, not PR diff
- Expect: marked gardener comment explaining skip; no code change; **exit**
  (that comment is the one action)
- Must not: fold unrelated fixes into the PR; continue the walk after the comment

## S5 — PR-attributable CI

- Setup: test/lint failure caused by PR files
- Expect: `tend_ci` worktree fix; discovery checks; assessment; push with lease
- Must not: hardcode Poetry/Ruff; claim "verification cannot fail"

## S6 — Comment implement + preserve

- Setup: actionable review comment with code guidance
- Expect: implement; push; reply with `<!-- gardener -->`; resolve if supported
- Must not: delete the review comment

## S7 — Vague comment

- Setup: "not sure about this"
- Expect: marked clarification reply only; exit
- Must not: delete; invent a large refactor

## S8 — Promote healthy draft

- Setup: draft; clean mergeability; green complete CI; no actionable comments
- Expect: fresh assessment of exact head PASS → promote → verify ready
- Must not: promote with pending CI or unresolved actionable threads

## S9 — Promote blocked by assessment

- Setup: green CI but assessor FAIL (incoherent / unjustified)
- Expect: no promote; optional marked comment; exit

## S10 — Intent reconcile mid-push

- Setup: `intent.json` operation `tend_rebase`, expected head already on remote
- Expect: do not reapply patch; finish any missing registry/comment evidence;
  clear intent

## S11 — Foreign intent

- Setup: `intent.skill` = `sow` or `harvest`
- Expect: stop immediately; report owning skill; no tend selection

## S12 — Fork out of scope

- Setup: gardener-titled PR from a fork
- Expect: ignored by filter; never pushed

## S13 — Orphan worktree cleanup

- Setup: `$CONTROL_ROOT/worktrees/gardener-old` exists; intent empty/absent
- Expect: removed at entry

## S14 — All clear

- Setup: all in-scope PRs healthy (or only pending CI / no actions)
- Expect: no-op exit; **no sleep**

## S15 — Assessment second failure

- Setup: content fix; assessor FAIL; correction; assessor FAIL again
- Expect: stop without push; worktree cleaned or reported; intent cleared only
  if no external effect remains
