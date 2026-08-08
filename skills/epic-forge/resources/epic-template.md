# Epic Body Template

Use this structure for every epic. Replace `{placeholders}`. Keep ALL detail inline — never reference local docs, gitignored files, or external repos. The epic body is the normative contract for the work it tracks.

```markdown
# Epic: {Title}

**Sibling epic**: {SIBLING_EPIC}   (omit if standalone; replaced with `repo#number` in the annotation pass)

## Goal

{One paragraph: the outcome this epic delivers. Testable, concrete.}

## Context

{Why this exists. Prior decisions, constraints, the problem being solved. Reference
other epics/issues by number where relevant. No external file references.}

## {Protocol|Architecture} specification (normative — inline, no external references)

{The full contract this epic implements: message catalog, DTOs, addressing, policies,
data models, endpoints, wire formats. Every detail an implementer needs, inline.}

## Phases & tasks

Each row is one issue. **Priority**: P0 = critical-path foundation; P1 = high; P2 = medium; P3 = polish.
**Dependencies** = issues that must merge first (recorded as native `blocked_by` links
in the linking pass). The authoritative state of each issue is its forge state
(open/closed/merged) — not this table.

| # | Issue | Phase | P | Depends on |
|---|-------|-------|---|------------|
| {P}-1 | {title} | {phase} | P0 | — |

## Critical path

{ASCII DAG of the longest must-land-in-sequence path}

## Cross-repo readiness gate

{Which sibling-epic issues must land before this epic's issues can be end-to-end
verified. Cross-repo deps via native blocked_by (database ids) + markdown hyperlinks.}

## Out of scope

- {what this epic explicitly does NOT cover}

## Acceptance criteria (Done when)

- [ ] {testable completion criterion}
```

## Rules

- The authoritative state of each issue is its forge state (open/closed/merged) — never the task table.
- Every issue in the table is created as a separate forge issue whose body references this epic (`Epic: #N`).
- Prefix per epic (e.g. `D-`, `B-`, `A-`, `W-`); detect collisions with existing issues before choosing.
- Cross-repo sibling epics reference each other by number (`**Sibling epic**: repo#n`).
