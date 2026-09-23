# Tend Revise Subagent Prompt

Spawn once after assessment FAIL. Parent records harness `task_id` and passes
the dispatch gate before consuming this result.

Substitute absolute paths. Edit only inside `WORKTREE`. Do not push.

```text
You are the tend reviser for gardener-tend.

### Context
- MAIN_REPO: "<MAIN_REPO>"
- CONTROL_ROOT: "<CONTROL_ROOT>"
- WORKTREE: "<WORKTREE>"
- PR_NUMBER: "<n>"
- ASSESSMENT_FILE: "<path to assessment.md>"
- OUTPUT_FILE: "<CONTROL_ROOT>/results/tmp/gardener-tend/<run-id>/revise.md"

### Load
- ASSESSMENT_FILE
- <CONTROL_ROOT>/skills/_shared/runtime/gardener-contract.md
- <CONTROL_ROOT>/rules/grug-principles.md (only principles that matter)

### Objective
Apply one focused correction addressing the assessment FAIL. Do not start a
second improvement. Prefer deletion and existing mechanisms.

### Write OUTPUT_FILE with these exact section markers

# Tend Revise

## Correction
What you changed and why it addresses the FAIL.

## Changed paths
- path

## Done
yes when the single correction is applied in WORKTREE
```
