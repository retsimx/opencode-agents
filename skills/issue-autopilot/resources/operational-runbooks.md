# Operational Runbooks for `issue-autopilot`

This resource provides execution runbooks and failure recovery protocols for orchestrators running `issue-autopilot`.

---

## 1. Worktree Isolation & Path Anchoring Runbook (Guardrails 3 & 4)

### Purpose
To ensure all code modifications occur in an isolated git worktree while all audit trails, designs, plans, and results persist in the parent repository across worktree lifecycles.

### Canonical Path Setup (Phase 0)
```bash
# 1. Resolve parent repository root
PARENT_REPO=$(git rev-parse --show-toplevel)

# 2. Define isolated worktree destination
WORKTREE=$(realpath ../<repo-name>-<number>)

# 3. Define persistent results directory
RESULTS_DIR="${PARENT_REPO}/.agents/results"
mkdir -p "${RESULTS_DIR}"
```

### Worktree Creation
```bash
git fetch origin main
git checkout main
git pull origin main
git worktree add "$WORKTREE" -b "$BRANCH" main
```

### Worktree Preservation Invariant (Guardrail 4)
If ANY phase encounters an unrecoverable error, gate failure, or is blocked:
- **STRICTLY FORBIDDEN**: Running `git worktree remove "$WORKTREE"`.
- **MANDATORY**: Leave `$WORKTREE` intact on disk for developer reproduction and debugging.
- Only successful completion of Phase 4 worktree teardown or explicit user instruction authorizes removal.

---

## 2. Fast-Fail Local CI Sanity Runbook (Guardrail 2)

### Purpose
To verify that no syntax errors, lint defects, type mismatches, or test regressions were introduced during implementation and refinement before git commits or PRs are created.

### Execution Sequence (Phase 4 Step 1)
```bash
cd "$WORKTREE"

# 1. Static analysis / Linting / Formatting
npm run lint 2>/dev/null || ruff check . 2>/dev/null || cargo clippy 2>/dev/null || true

# 2. Type Checking
tsc --noEmit 2>/dev/null || mypy . 2>/dev/null || true

# 3. Test Suite
npm test 2>/dev/null || pytest 2>/dev/null || cargo test 2>/dev/null || go test ./... 2>/dev/null
```

### Binary Gate & Remediation
- **Pass (Exit Code 0)**: Proceed to Phase 4 Step 2 (Commit & PR generation).
- **Fail (Exit Code != 0)**:
  1. If failures are trivial auto-formattable issues (e.g. `prettier`, `ruff format`), apply them safely.
  2. For non-trivial errors or test failures, dispatch a Debug Subagent (`.agents/skills/issue-autopilot/resources/subagent-prompts.md#3-fast-fail-local-ci-remediation-debug-subagent-prompt`).
  3. Re-run sanity checks.
  4. If sanity checks still fail: Halt immediately, preserve worktree intact (Guardrail 4), and alert user in chat.

---

## 3. Forge PR Creation & Provider CLI Fallback Runbook

### Purpose
Standardized commands for opening draft PRs and fallback procedures when automated forge operations encounter rate limits or network issues.

### Provider Draft PR Commands
- **GitHub (`gh`)**:
  ```bash
  gh pr create --draft --base main --title "<title>" --body-file /tmp/pr-body.txt
  ```
- **GitLab (`glab`)**:
  ```bash
  glab mr create --draft --target-branch main --title "<title>" --description "$(cat /tmp/pr-body.txt)"
  ```

### Manual Fallback Instructions (If CLI Fails)
If PR creation fails due to missing permissions or CLI errors, provide manual commands to the user:
```bash
# Push branch manually
git push -u origin "$BRANCH"

# Open PR via web UI or CLI:
# GitHub:
gh pr create --draft --base main --head "$BRANCH"
# GitLab:
glab mr create --draft --target-branch main --source-branch "$BRANCH"
```
