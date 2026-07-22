# Gardener — Hard Exclusions

These paths must NEVER be read for modification, staged, or committed.

## Auto-generated

- `**/migrations/*.py` (except `__init__.py` if needed for package structure — do not edit migration bodies)

## Vendored / compiled assets

- `static/**/*.min.js`
- `static/**/*.bundle.*`
- Compiled CSS bundles (e.g. `*.min.css` under `static/`)

## Build / cache / tooling output

- `.venv/`
- `node_modules/`
- `coverage/`
- `test_output/`
- `htmlcov/`
- `__pycache__/`
- `*.pyc`

## Secrets

- `.env`
- `.env.*`
- `credentials.json`
- `*.pem`, `*.key`
- Any file containing API keys, tokens, or passwords

## Enforcement

Before staging, run `git diff --name-only` and reject the iteration if any changed file matches an exclusion pattern. Return `EXCLUDED:<path>` to the orchestrator.
