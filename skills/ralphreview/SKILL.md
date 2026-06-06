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
Iteratively review and remediate the current implementation until review stability is achieved (three consecutive clean reviews where no new *fixable* issues are found).

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
- A codebase with changes to review
- An optional scope string (DIFF, COMMIT:<hash>, FILES:<list>, FEATURE:<desc>, etc.)
- A `clean_review_streak` variable initialized to 0

### Expected outputs
- Zero relevant code issues after the loop exits
- All identified issues remediated
- A `.ralphreview-state-<hash>` file documenting every finding and its disposition

### Dependencies
- OpenCode `task` tool — for delegating review and remediation
- `deep-review` skill — performs the actual deep code review inside the Task subagent

### Control-flow features
- Loops until 3 consecutive clean reviews or failure escalation
- Branches by presence of new issues in review results
- Any fix resets the clean streak to 0
- Orchestration only — never performs review or remediation directly
- Session-isolated via unique state file per invocation

## Structural Flow

### Entry
1. If scope was not provided by the user, ask: "What scope should I review? Options: DIFF (uncommitted changes), COMMIT:<hash>, COMMIT:<r1>..<r2>, FILES:<file1,file2>, FEATURE:<description>, or leave blank for full project scan."
2. Accept the answer and store as `SCOPE`.
3. Generate a short (3–4 character) unique identifier — e.g. via `date +%s | tail -c 5` or `openssl rand -hex 2`.
4. Define `STATE_FILE = .ralphreview-state-<unique_id>`.
5. Initialize `clean_review_streak := 0`.
6. Create/truncate `STATE_FILE` with a header line:
   ```
   # RalphReview State — STATUS|SEVERITY|file:line|description
   # Statuses: NEW, FIXED, SKIPPED
   ```

### Scenes
1. **PREPARE**: Initialize state and ask for scope if needed.
2. **REVIEW**: Invoke a Task subagent with the `deep-review` skill. The subagent reads `STATE_FILE` to dedup, runs deep-review, **appends NEW findings to `STATE_FILE`**, and returns only a signal (`NEW_FOUND` or `NO_NEW_FINDINGS`).
3. **EVALUATE**: If `NO_NEW_FINDINGS`, increment streak and loop. Otherwise invoke remediation.
4. **REMEDIATE**: Invoke a Task subagent that reads `STATE_FILE` (finds `NEW` entries), evaluates each finding, fixes FIXABLE ones, **appends FIXED/SKIPPED entries to `STATE_FILE`**, and returns ONLY what was fixed.
5. **STREAK**: If anything was fixed, reset streak to 0. If all were skipped (or no findings), increment streak.
6. **FINALIZE**: Exit when streak == 3. Optionally read `STATE_FILE` and report a summary to the user.

### Transitions
- If new findings exist: send to remediation for triage.
- If remediation fixed anything: reset streak, loop.
- If remediation skipped everything (all intentional / not practical): increment streak, loop. They're in STATE_FILE and won't be re-flagged.
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
| Ask for scope | `REQUEST` | Clarification phase |
| Generate state file path | `UPDATE_STATE` | Unique session ID |
| Invoke review | `CALL_TOOL` | OpenCode `task` with STATE_FILE path; returns NEW_FOUND / NO_NEW_FINDINGS |
| Inspect review signal | `READ` | Boolean signal from `task` return |
| Invoke remediation | `CALL_TOOL` | OpenCode `task` with STATE_FILE path; remediator reads NEW entries from file |
| Parse remediation result | `READ` | Extract fixed list from Task output |
| Update streak | `UPDATE_STATE` | Reset if fixed found, increment if all skipped/none |
| Read summary | `READ` | Optional STATE_FILE read on exit |
| Report result | `NOTIFY` | Final completion or escalation message |

### Tools and instruments
- OpenCode `task` tool (`subagent_type="general"`) — for delegating review and remediation.
  **IMPORTANT: `task` is a built-in agent tool, NOT a shell command. Invoke it like `read`, `write`, or `grep` — through your tool-calling interface. Never run `$ task ...` in bash.**

### Canonical workflow path

**YOU MUST follow this exact sequence. Do not deviate.**

**CRITICAL: The `task` tool is a built-in agent function call, the same as `read`, `write`, `grep`. Do NOT run it as a bash command — use your tool-calling interface directly.**

```
# Phase 0: Entry ────────────────────────────────────────────
SCOPE = <from user or default to "DIFF">
STATE_FILE = .ralphreview-state-$(date +%s | tail -c 5)
clean_review_streak = 0
Create STATE_FILE with header

while clean_review_streak < 3:

    # ── Phase A: Review ──────────────────────────────────────
    # Do NOT read source files or run git diff. Delegate via
    # the built-in `task` tool.

    Call the `task` tool with these parameters:
      subagent_type: "general"
      description: "deep-review"
      prompt:
        "Load the deep-review skill and review scope: <SCOPE>.

         READ <STATE_FILE> first. Any issue already listed in that
         file (regardless of status) must NOT be re-reported. Only
         report genuinely NEW issues not present in the state file.

         APPEND each new finding to <STATE_FILE> in this format:
           NEW|<SEVERITY>|<file:line>|<description>

         If you appended any findings, return exactly: NEW_FOUND
         If no new findings, return exactly:     NO_NEW_FINDINGS"

    Wait for the subagent to return.
    signal = result  # either "NEW_FOUND" or "NO_NEW_FINDINGS"

    # ── Phase B: Evaluate ─────────────────────────────────────
    if signal == "NO_NEW_FINDINGS":
        clean_review_streak += 1
        continue  # back to Phase A

    # ── Phase C: Remediation ──────────────────────────────────
    # Do NOT read source files or edit files yourself. Delegate
    # via the built-in `task` tool.

    Call the `task` tool with these parameters:
      subagent_type: "general"
      description: "remediation"
      prompt:
        "Read <STATE_FILE>. Find all entries with status NEW.

         For each NEW entry, examine the actual code and determine:
           - FIXABLE — a real issue that should be fixed now
           - INTENTIONAL — by design or conscious tradeoff
           - NOT PRACTICAL — would require disproportionate effort

         Fix FIXABLE ones. Leave INTENTIONAL and NOT PRACTICAL alone.

         APPEND each finding to <STATE_FILE> in this format:
           FIXED|<SEVERITY>|<file:line>|<description>
           SKIPPED|<SEVERITY>|<file:line>|<description> — reason: <why>

         Do NOT re-write the entire file — only APPEND new lines.
         Do NOT remove or modify the existing NEW lines.

         Return ONLY what was FIXED, one per line:
           FIXED|<SEVERITY>|<file:line>|<description>

         If nothing was fixed, return exactly: NO_FIXES"

    Wait for remediation to complete.
    result = output

    # ── Phase D: Streak Update ────────────────────────────────
    if result == "NO_FIXES":
        clean_review_streak += 1  # all skipped = stable iteration
    else:
        clean_review_streak = 0   # something was fixed

    # Loop back to Phase A

exit  # clean_review_streak == 3
```

### Resource scope
| Scope | Resource target |
|-------|-----------------|
| `LOCAL_FS` | Working tree git state |
| `PROCESS` | `task` tool subagent processes |
| `MEMORY` | `clean_review_streak`, `STATE_FILE` path |
| `LOCAL_FS` | `STATE_FILE` on disk |

### Preconditions
- Codebase has changes in the requested scope.
- `deep-review` skill is available at `.agents/skills/deep-review/SKILL.md`.

### Effects and side effects
- Invokes subagents via the `task` tool for review and remediation.
- May modify code through remediation agents.
- Creates/updates `STATE_FILE` with all findings and dispositions.
- Does not stage or commit changes.

### Guardrails
1. **Never perform your own review** — always delegate to a subagent via the built-in `task` tool.
2. **Never skip review** or substitute your own judgment.
3. **Review and remediation MUST use separate `task` tool invocations** — never combine them.
4. **Any fix resets `clean_review_streak` to 0.** If all findings are skipped (intentional / not practical), increment streak — they're in STATE_FILE and won't be re-flagged.
5. **Foreground context is orchestration only** — no review or remediation inline.
6. **Prefer minimal targeted fixes over broad refactors.**
7. **Exit immediately when `clean_review_streak == 3`.**
8. **Reviewer writes NEW findings to STATE_FILE.** The review subagent reads STATE_FILE for dedup, runs deep-review, and appends any NEW findings to STATE_FILE. It returns only `NEW_FOUND` or `NO_NEW_FINDINGS` — never the findings themselves.
9. **Remediation agent triages fixability.** The remediation subagent reads STATE_FILE to find NEW entries, examines the actual code, fixes FIXABLE ones, and appends FIXED/SKIPPED entries. Streak only resets when something was actually fixed.
10. **State file is the single source of truth.** The orchestrator does NOT maintain `handled_issues` in memory. All findings (NEW, FIXED, SKIPPED) live in STATE_FILE. The only orchestrator state is `clean_review_streak` and `STATE_FILE` path.
11. **Orchestrator never reads STATE_FILE during the loop.** The orchestrator only reads the task return signal (`NEW_FOUND` / `NO_NEW_FINDINGS` from reviewer; `FIXED|...` / `NO_FIXES` from remediator). It does NOT read the state file to check what findings exist — that's the subagents' job. The optional exit-time read for a summary is the sole exception.
12. **Every task invocation receives the STATE_FILE path** so subagents can read/write it. The path includes a unique session ID so concurrent ralphreview loops don't collide.
13. **TOOL RESTRICTION — Review: You MUST NOT read any source files in the foreground.** You MUST NOT run `git diff` (full diff). Your only tool for Phase A is the built-in `task` tool (used via your tool-calling interface, NOT bash). Do NOT try to run `$ task` as a shell command. All code understanding must come from the deep-review skill's output.
14. **TOOL RESTRICTION — Remediation: You MUST NOT read any source files to prepare or verify remediation.** You MUST NOT edit any files directly. You MUST NOT run `git diff` beyond `git diff --stat`. ALL remediation MUST go through a `task` tool invocation.
15. **TOOL RESTRICTION — Diff: You may only run `git diff --stat` (file names only) to check whether files changed. Never the full diff.**
16. **CRITICAL: The `task` tool is a built-in agent function call, the same as `read`, `write`, or `grep`. It is NOT a bash command. Never run `$ task ...` in a shell — use the agent's tool-calling interface instead.**

## References

- Deep-review skill: `.agents/skills/deep-review/SKILL.md`
