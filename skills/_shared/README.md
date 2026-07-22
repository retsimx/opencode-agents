# Shared Resource Layout

`_shared/` contains cross-skill resources grouped by loading behavior and ownership.

## Path convention (host project root)

This library is cloned as `.agents/` inside a host project. Agents run with **cwd = host project root**. All load/read/write paths in skills MUST be written from that root:

| Kind | Format |
|------|--------|
| Shared doc | `.agents/skills/_shared/{core\|conditional\|runtime}/file.md` |
| Skill doc | `.agents/skills/{skill-name}/SKILL.md` |
| Skill resource | `.agents/skills/{skill-name}/resources/file.md` |
| Session artifacts | `.agents/results/...` |
| Host project docs | `AGENTS.md`, `docs/...` (no `.agents/` prefix) |

Do **not** use skill-relative paths like `../_shared/...` or bare filenames like `providers.md` for agent tool operations.

## Structure

- `core/`
  - Always-on or commonly referenced rules and guides.
  - Examples: context loading, clarification, difficulty, reasoning, lessons learned.
- `conditional/`
  - Load only when the workflow reaches a specific trigger.
  - Examples: quality score, experiment ledger, exploration loop.
- `runtime/`
  - Runtime-injected or CLI-specific protocols.
  - Examples: coordination protocol, execution protocol, forge provider map (`.agents/skills/_shared/runtime/providers.md`), gardener outer-loop recipes (`.agents/skills/_shared/runtime/gardener-running.md`).

## Workflow-Owned Resources

Workflow-specific materials do not belong in `_shared/`.

- `.agents/skills/ultrawork/resources/phase-gates.md`
- `.agents/skills/ultrawork/resources/multi-review-protocol.md`

## Load Classes

- `always`: load at task start or during normal execution
- `conditional`: load only on the documented trigger
- `runtime-injected`: supplied automatically by CLI/runtime code
- `workflow-only`: owned by a single workflow, not shared across all skills
