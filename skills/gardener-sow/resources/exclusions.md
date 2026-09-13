# Gardener Sow — Path Exclusions

Never read for modification, stage, or commit paths that are secrets,
generated, or vendored. Prefer repository guidance when it names additional
off-limits or generated paths.

## Secrets

- `.env`, `.env.*`
- `credentials.json`, `*.pem`, `*.key`
- Files that clearly contain API keys, tokens, or passwords

## Generated / vendored / build output

- `.venv/`, `node_modules/`, `vendor/` (when third-party trees)
- `__pycache__/`, `*.pyc`, `*.pyo`
- `coverage/`, `htmlcov/`, `test_output/`, `dist/`, `build/`
- Minified or bundled static artifacts (`*.min.js`, `*.min.css`, `*.bundle.*`)

## Enforcement

Before staging: `git diff --name-only`. If any changed path matches, abort the
ship/implement step and report the path. Do not invent framework-specific
exclusions (e.g. app migrations) unless the target repository marks them
generated or off-limits.
