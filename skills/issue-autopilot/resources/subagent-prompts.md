# Subagent Prompt Templates for `issue-autopilot`

This resource provides standardized prompt templates for Task subagents dispatched during `issue-autopilot` execution. All subagents operate under **Zero-Context Relay** (pass-by-reference) and the **Universal File-First State I/O Architecture**.

---

## 1. SCM Specialist Subagent Prompt (Phase 4 Step 2)

Dispatched via the subagent tool to generate Conventional Commit messages and draft PR descriptions based on worktree diffs.

```markdown
Role: "SCM Specialist Agent"
Prompt:
Load the scm skill (.agents/skills/scm/SKILL.md) and read the Conventional Commits section.
WORKTREE: <$WORKTREE>
RESULTS_DIR: <$RESULTS_DIR>
SESSION_ID: <sessionId>
OUTPUT_FILE: <$RESULTS_DIR>/result-scm-ship-<sessionId>.md

1. Inspect changes in WORKTREE using `git diff --stat` and `git diff`.
2. Determine the Conventional Commit type and scope (e.g. feat(scope), fix(scope)).
3. Write a commit message:
   - Imperative mood, lowercase, <=72 chars title, no trailing period.
   - Follow project AGENTS.md commit style if it exists.
   - Body: concise bullet points describing changes (no verbose numbers/filenames).
   - Issue reference: Closes #<number> (or Fixes #<number>)
4. Write a PR/MR body:
   - Bullet points summarizing what was implemented and verified.
   - Reference: Closes #<number>
5. Write the commit message to /tmp/commit-msg.txt and the PR body to /tmp/pr-body.txt.
6. Write technical summary to OUTPUT_FILE and verify it exists on disk.

Return ONLY the standard 4-line chat completion summary:
### Task Complete: SCM Specialist Agent — Commit & PR Preparation
- **Status**: SUCCESS | BLOCKED | FAILED
- **Summary**:
  - Title: <commit title>
  - Type/Scope: <type>(<scope>)
  - Artifacts: /tmp/commit-msg.txt, /tmp/pr-body.txt
- **Artifact**: `file:///<OUTPUT_FILE>`
```

---

## 2. Issue Communicator Subagent Prompt (Phase 5 Step 1)

Dispatched via the subagent tool to synthesize non-technical release notes and post a plain-English comment to the forge issue.

```markdown
Role: "Issue Communicator Agent"
Prompt:
Write and post a plain-English summary comment on issue #<number>.

PROVIDER: <github|gitlab>
PARENT_REPO: <$PARENT_REPO>
BRANCH: <branch-name>
ISSUE_NUMBER: <number>
PR_URL: <pr-url>
RESULTS_DIR: <$RESULTS_DIR>
SESSION_ID: <sessionId>
OUTPUT_FILE: <$RESULTS_DIR>/result-issue-comment-<sessionId>.md

Read .agents/skills/_shared/runtime/providers.md and use ONLY the Comment on issue command for PROVIDER.

First, inspect the merged/pushed diff from PARENT_REPO:
  cd <PARENT_REPO>
  git diff origin/main..<BRANCH>

Then write a comment following these strict rules:
1. NO code references — no file names, function names, line numbers, variable names, or implementation details.
2. NO technical jargon — describe technical concepts in everyday language.
3. COVER:
   - What was wrong (restate the issue in plain terms).
   - Decisions made and why.
   - What was done, in plain language.
   - Behavior changes or improvements the user will notice.
4. TONE: helpful, transparent, humble, professional.
5. FORMAT: short paragraphs, no markdown headings, no bullet lists.

Fetch the issue for context using the View issue command for PROVIDER.
Write the comment to /tmp/issue-comment.txt and post it with the Comment on issue command for PROVIDER:
  GitHub: gh issue comment <ISSUE_NUMBER> --body-file /tmp/issue-comment.txt
  GitLab: glab issue note <ISSUE_NUMBER> --message "$(cat /tmp/issue-comment.txt)"

Write technical summary to OUTPUT_FILE and verify it exists on disk.
Return ONLY the standard 4-line chat completion summary with the posted comment text in the summary bullets:
### Task Complete: Issue Communicator Agent — Issue Comment
- **Status**: SUCCESS | BLOCKED | FAILED
- **Summary**:
  - {Summary point 1 / Comment excerpt}
  - {Summary point 2}
  - {Summary point 3}
- **Artifact**: `file:///<OUTPUT_FILE>`
```

---

## 3. Fast-Fail Local CI Remediation Debug Subagent Prompt (Phase 4 Step 1 Fallback)

Dispatched via the subagent tool if local CI sanity checks fail prior to committing.

```markdown
Role: "Debug Agent — CI Remediation"
Prompt:
Remediate local CI sanity failures in $WORKTREE prior to commit/PR.

WORKTREE: <$WORKTREE>
RESULTS_DIR: <$RESULTS_DIR>
SESSION_ID: <sessionId>
OUTPUT_FILE: <$RESULTS_DIR>/result-debug-ci-<sessionId>.md
FAILURE_LOGS: <failure logs or error output>

1. Inspect failures in WORKTREE:
   - Identify syntax errors, type errors, linting issues, or broken unit tests.
2. Apply root-cause remediation directly to files in WORKTREE.
   - Forbid tactical patches or suppression comments unless strictly justified.
3. Re-run local CI test and lint commands to verify 100% pass (Exit Code 0).
4. Write remediation log to OUTPUT_FILE and verify on disk.

Return ONLY the standard 4-line chat completion summary:
### Task Complete: Debug Agent — CI Remediation
- **Status**: SUCCESS | BLOCKED | FAILED
- **Summary**:
  - Root Cause: <root cause of failure>
  - Fix Applied: <remediation details>
  - Verification: <test/lint result>
- **Artifact**: `file:///<OUTPUT_FILE>`
```
