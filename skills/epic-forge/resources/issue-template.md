# Issue Body Template

Use this structure for every issue. Replace `{placeholders}`. Keep ALL detail inline — never reference local docs, gitignored files, or external repos. An issue must be copy-pasteable into a fresh session with everything needed to complete it unambiguously.

```markdown
# {P}-{n}: {Title}

**Epic**: #{epic-number}
**Phase**: {phase}

## Goal

{One paragraph: what this issue delivers. Testable.}

## Context

{Why this exists, relevant prior decisions, and the specific protocol/architecture
detail from the epic this issue implements — inline, not referenced.}

## Protocol contract (normative — inline)

{The exact message shapes, DTOs, wire formats, policies, or endpoint contracts this
issue must implement, copied inline from the epic. Every field, every enum, every
range. No ambiguity.}

## Out of scope

- {what this issue explicitly does NOT do}

## Files to modify

- `{path}` — {what changes}
- `{path}` — {what changes}

## Acceptance criteria

- [ ] {testable criterion}
- [ ] {testable criterion}

## Dependencies

- **Depends on**: {P}-{n} (and any cross-repo: {repo}#{n}).
- **Blocks**: {P}-{n}.

## Implementation notes

- {guidance, edge cases, ordering, gotchas}

## Test plan

- {unit/integration tests covering the acceptance criteria}
```

## Rules

- **Self-contained**: all protocol detail inline; a fresh session can complete the issue without reading anything else.
- **No local-docs references**: never cite `docs/`, gitignored files, or external repos as the source of detail.
- **Test plan required**: every issue has one; security/testing are part of the task, not a separate phase.
- **Acceptance criteria are checkboxes**: testable, measurable, unambiguous.
- **Dependencies** reference issue numbers (prefix + number) and cross-repo issues as `repo#number`.
