---
name: ralphreview
description: >
  Iterative review-and-remediate agent that runs the deep-review skill in a loop
  until three consecutive clean reviews pass. Orchestration-only — never performs
  review or remediation directly.
---

# Ralph Review — Iterative QA Loop

## Scheduling

### Goal
Iteratively review and remediate the current implementation until review stability is achieved (three consecutive clean reviews where no new issues are found).

### Intent signature
- User invokes `/ralph` to run the ralph review loop
- User wants an automated review loop that drives toward clean reviews
- QA gate before shipping

### When to use
- QA gate before shipping
- Automated review loop without manual re-invocation
- Any scenario needing verified quality before merge

### When NOT to use
- Performing a one-shot review without the loop -> use `deep-review` skill
- Debugging a specific bug -> use `debug`

### Expected inputs
- A codebase with uncommitted changes to review
- A `clean_review_streak` variable initialized to 0
- A `handled_issues` list initialized to empty

### Expected outputs
- Zero relevant code issues after the loop exits
- All identified issues remediated

### Dependencies
- OpenCode `task` tool — for delegating review and remediation
- `deep-review` skill — performs the actual deep code review inside the Task subagent

### Control-flow features
- Loops until 3 consecutive clean reviews or failure escalation
- Branches by presence of new issues in review results
- Any fix resets the clean streak to 0
- Orchestration only — never performs review or remediation directly

## Structural Flow

### Entry
1. Confirm the codebase has uncommitted changes to review.
2. Initialize `clean_review_streak := 0`.
3. Initialize `handled_issues := []`.

### Scenes
1. **PREPARE**: Initialize state.
2. **REVIEW**: Invoke a Task subagent with the `deep-review` skill, passing `handled_issues` as context so it does not re-report already-handled issues.
3. **EVALUATE**: Dedup against `handled_issues`. If new findings remain, invoke a remediation Task. The remediation Task returns which findings were fixed and which were skipped (intentional / not practical).
4. **RECOVER**: Append ALL findings (fixed + skipped) to `handled_issues`. If any were fixed, reset streak to 0. Loop back to REVIEW.
5. **FINALIZE**: If no new findings, increment streak. Exit when streak == 3.

### Transitions
- If new findings exist: send to remediation for triage.
- If remediation fixed anything: reset streak, loop.
- If remediation skipped everything (all intentional / not practical): increment streak, loop. They're in handled_issues and won't be re-flagged.
- If no new findings: increment streak, loop or exit at 3.
- If remediation introduces issues, next REVIEW catches them.

### Failure and recovery
| Failure | Recovery |
|---------|----------|
| Review returns ambiguous results | Re-invoke review with more specific scope |
| Remediation introduces new issues | Next loop catches them |
| Loop exceeds reasonable iterations | Report partial results to user, exit with warning |

### Exit
- Success: `clean_review_streak == 3`
- Partial success: streak never reached, user intervention needed
- Failure: `task` tool unavailable

## Logical Operations

### Actions
| Action | SSL primitive | Evidence |
|--------|---------------|----------|
| Invoke review | `CALL_TOOL` | OpenCode `task` tool with review instructions + handled_issues |
| Inspect results | `READ` | `task` tool return value |
| Dedup against handled | `COMPARE` | Filter findings not already in handled_issues |
| Invoke remediation | `CALL_TOOL` | OpenCode `task` tool with findings; returns fixed + skipped |
| Parse remediation result | `READ` | Extract fixed vs skipped from Task output |
| Append to context | `UPDATE_STATE` | Add ALL findings (fixed + skipped) to handled_issues |
| Update streak | `UPDATE_STATE` | Reset if fixed found, unchanged if all skipped |
| Report result | `NOTIFY` | Final completion or escalation message |

### Tools and instruments
- OpenCode `task` tool (`subagent_type="general"`) — for delegating review and remediation

### Canonical workflow path

**YOU MUST follow this exact sequence. Do not deviate.**

```
clean_review_streak := 0
handled_issues := []

while clean_review_streak < 3:

    # ── Phase A: Review ──────────────────────────────────────────
    # Use the `task` tool. Do NOT read files or run git diff yourself.

    Use the `task` tool (subagent_type="general") with this exact prompt:

        "Load the deep-review skill and review the recent uncommitted
         changes in this project. Provide scope as: DIFF (uncommitted
         changes). Return findings in the format specified by the
         deep-review skill.

         Previously handled issues — do NOT re-report these:
         <handled_issues>"

    Wait for the Task subagent to return.
    Collect findings.

    # ── Phase B: Evaluate ─────────────────────────────────────────
    # If deep-review returned nothing, this iteration is clean.
    # If it returned findings, compare against handled_issues to
    # filter out anything already dealt with.

    new_findings = [f for f in findings if f not in handled_issues]

    if new_findings:

        # ── Phase C: Remediation ─────────────────────────────────
        # YOU MUST use the `task` tool here. Do NOT read any source
        # files. Do NOT edit any files. ALL remediation must be
        # delegated to this Task.
        #
        # The remediation Task evaluates each finding in context
        # and reports back which were fixed vs skipped (intentional,
        # not practical, by design, etc.).

        Use the `task` tool (subagent_type="general") with this exact prompt:

            "Review the following findings against the actual code.
             For each one, determine if it is:
               - FIXABLE — a real issue that should be fixed now
               - INTENTIONAL — by design or a conscious tradeoff
               - NOT PRACTICAL — would require disproportionate effort

             Fix the FIXABLE ones. Leave INTENTIONAL and NOT PRACTICAL
             alone. Then report back in this exact format:

             Fixed:
               - <finding description> (<file:line>)
             Skipped:
               - <finding description> (<file:line>) — reason: <why>

             Findings to evaluate:
             <list new_findings>"

        Wait for remediation to complete.
        Parse the result to extract fixed_findings and skipped_findings.

        for f in fixed_findings + skipped_findings:
            handled_issues.append(f)

        if fixed_findings:
            clean_review_streak := 0
        else:
            # All were skipped — remediation confirmed they're not
            # actionable (intentional / not practical). They're in
            # handled_issues so deep-review won't re-flag them.
            # Count this as a clean iteration.
            clean_review_streak += 1

        # Loop back to Phase A.

    else:
        clean_review_streak += 1
        # Loop back to Phase A.

exit  # clean_review_streak == 3
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `LOCAL_FS` | Working tree git state |
| `PROCESS` | `task` tool subagent processes |
| `MEMORY` | `clean_review_streak`, `handled_issues` |

### Preconditions
- Codebase has uncommitted changes.
- `deep-review` skill is available at `.agents/skills/deep-review/SKILL.md`.

### Effects and side effects
- Invokes subagents via the `task` tool for review and remediation.
- May modify code through remediation agents.
- Does not stage or commit changes.

### Guardrails
1. **Never perform your own review** — always delegate to a Task subagent with the `deep-review` skill.
2. **Never skip review** or substitute your own judgment.
3. **Review and remediation MUST use SEPARATE Task invocations** — never combine them.
4. **Any fix resets `clean_review_streak` to 0.** If all findings are skipped (intentional / not practical), increment streak — they've been evaluated and won't be re-flagged.
5. **Foreground context is orchestration only** — no review or remediation inline.
6. **Prefer minimal targeted fixes over broad refactors.**
7. **Exit immediately when `clean_review_streak == 3`.**
8. **Remediation agent triages fixability.** The remediation Task reads the actual code and decides what to fix vs skip. INTENTIONAL (by design) and NOT PRACTICAL (disproportionate effort) findings are skipped and added to `handled_issues` — they won't be re-flagged. Streak only resets when something was actually fixed.
9. **Maintain `handled_issues` across iterations.** Include it in every review invocation so the deep-review skill knows what not to re-report. Append fixed findings after each remediation cycle.
10. **TOOL RESTRICTION — Review: You MUST NOT read any source files in the foreground.** You MUST NOT run `git diff` (full diff). Your only tools for Phase A are the `task` tool (to delegate) and reading the Task's returned findings. All code understanding must come from the deep-review skill's output. You do NOT evaluate fixability yourself — that is the remediation Task's job.
11. **TOOL RESTRICTION — Remediation: You MUST NOT read any source files to prepare or verify remediation.** You MUST NOT edit any files directly. You MUST NOT run `git diff` beyond `git diff --stat`. ALL remediation MUST go through a Task subagent.
12. **TOOL RESTRICTION — Diff: You may only run `git diff --stat` (file names only) to check whether files changed. Never the full diff.**

## References

- Deep-review skill: `.agents/skills/deep-review/SKILL.md`
