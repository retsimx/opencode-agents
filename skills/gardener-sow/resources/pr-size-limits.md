# Gardener — PR Size Limits

Every iteration must produce a **human-reviewable** draft PR. One specific concern, minimal diff.

## Hard limits

| Metric | Limit |
|--------|-------|
| Total diff lines | ≤ 150 (`git diff --stat` insertions + deletions) |
| Commits | Exactly 1 |
| Concerns | Exactly 1 — one bug, one smell, one lint fix, one coverage gap, etc. |

No hard file-count limit. Many files with tiny, related changes (e.g. renaming a constant across 6 files, one line each) are valid if total diff lines and single-concern rules are met.

## Soft guidance (not hard limits)

- Prefer fewer files when the same outcome is achievable
- Each changed file should be **necessary** for the single concern — no drive-by edits
- If diff is small but files are numerous, rationale must explain why (e.g. "constant used in 6 modules")

## What counts as one concern

Valid (single concern):

- Fix one lint violation in one module
- Add tests for one uncovered function
- Remove one dead code path
- Fix one specific bug with a minimal patch
- Rename one misleading variable or constant across all call sites
- Extract one helper from one oversized function (without changing callers elsewhere)

Invalid (too broad — reject or split):

- "Refactor the views module"
- "Fix all ruff errors"
- "Improve test coverage across tutoring/"
- "Update dependencies" (unless exactly one pin with one-line rationale)
- Touching unrelated files for convenience

## Enforcement points

1. **SCAN** — only propose items completable within limits; note estimated diff size in rationale.
2. **WORK** — before returning SUCCESS, run `git diff --stat`. If over 150 diff lines, narrow the fix or return `FAIL|too_large:<lines>N`.
3. **VERIFY** — re-check `git diff --stat`; return `FAIL|too_large:<lines>N` if exceeded.

## PR/MR best practice (SHIP subagent)

Draft PR body must include:

```markdown
## Summary
- One sentence: what changed and why

## Scope
- Single concern: <specific item>
- Files: <list>

## Test plan
- [ ] <specific test command or manual check>

## Reviewer notes
- No migrations, no vendored assets, no unrelated changes
```

Title: `chore(gardener): <specific description>` — imperative, under 72 chars.

Commit message matches title. No drive-by edits.
