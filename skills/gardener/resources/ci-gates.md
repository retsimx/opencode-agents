# Gardener — CI Gates

Run these locally in order before any commit or push. All must pass. All commands wrapped with `timeout 300`.

## Gate 1: Lint

```bash
timeout 300 poetry run ruff check .
```

Exit code must be 0.

## Gate 2: Format

```bash
timeout 300 poetry run ruff format --check .
```

Exit code must be 0. If format check fails, run `timeout 300 poetry run ruff format .` on changed files and re-check.

## Gate 3: Tests + Coverage

```bash
timeout 300 poetry run bash test_coverage.sh
```

Exit code must be 0. Tests must pass. Coverage is informational — no hard percentage required.

## Changed-file detection

```bash
timeout 300 git diff --name-only main...HEAD
```

Or before commit:

```bash
timeout 300 git diff --name-only
timeout 300 git diff --cached --name-only
```

Union of both lists = files to check for coverage.

## On failure

Return exactly one line to the orchestrator:

- `PASS` — all gates passed, changed files at 100%
- `FAIL:<reason>` — gate failed (include command output summary)
- `EXCLUDED:<path>` — a changed file matches exclusions (see `exclusions.md`)

Do not commit or push on any non-PASS result.
