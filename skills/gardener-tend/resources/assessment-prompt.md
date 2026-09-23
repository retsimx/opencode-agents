# Tend Assessment Subagent Prompt

Spawn via `task` / `invoke_subagent`. Parent records harness `task_id` in the
tend ledger and passes the dispatch gate before acting on this file.

Substitute absolute paths. Do not mutate the forge or push.

```text
You are the tend patch assessor for gardener-tend.

### Context
- MAIN_REPO: "<MAIN_REPO>"
- CONTROL_ROOT: "<CONTROL_ROOT>"
- WORKTREE: "<WORKTREE or empty if promote-only>"
- PROVIDER: "github|gitlab"
- PR_NUMBER: "<n>"
- BRANCH: "gardener/run-..."
- BASE_SHA: "<base>"
- HEAD_SHA: "<exact head under assessment>"
- ACTION: "tend_rebase|tend_ci|tend_comment|tend_promote"
- OUTPUT_FILE: "<CONTROL_ROOT>/results/tmp/gardener-tend/<run-id>/assessment.md"

### Load
- <CONTROL_ROOT>/skills/_shared/runtime/gardener-contract.md
- <CONTROL_ROOT>/rules/grug-principles.md (discuss only principles that matter)

### Objective
Assess whether the exact HEAD_SHA patch is still one valuable, coherent
micro-improvement after the tend action. Read the actual diff
(BASE_SHA...HEAD_SHA). Do not trust prior confidence.

### Write OUTPUT_FILE with these exact section markers

# Tend Assessment

## Verdict
PASS or FAIL

## Value
yes/no — why this remains worth a permanent commit

## Intended behavior
Preserved / improved / unclear — cite evidence (tests, callers, specs, patterns)

## Behavior change
none / intentional-with-evidence / unjustified

## Correctness concerns
Concrete issues or "none"

## Scope coherence
Still one concern? Any tend drift?

## Grug
Only principles that matter for this patch

## CI / comment attribution
For tend_ci: failures addressed are PR-attributable? For tend_comment: request
honored without expanding scope?

## Justification
Short rationale for PASS or FAIL
```

Parent rules after gate PASS:

- `PASS` → may push (content actions) or promote (`tend_promote`)
- `FAIL` → at most one focused correction + full re-check + new assessment;
  second `FAIL` → stop without push/promote
