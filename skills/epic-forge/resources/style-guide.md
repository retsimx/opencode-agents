# epic-forge Style Guide

The conventions every epic and issue must follow. These are non-negotiable — they are what make the contracts self-contained and unambiguous.

## Detail & self-containment

1. **No local-docs or gitignored references.** Never cite `docs/`, `.agents/`, gitignored files, or external repos as the source of detail. ALL detail is inline in the epic/issue bodies.
2. **Self-contained contracts.** Every issue is copy-pasteable into a fresh session and contains everything needed to complete it unambiguously — protocol shapes, DTOs, wire formats, ranges, enums, files, test plans.
3. **No ambiguity.** If a detail is unknown or unverified, state it explicitly (`unverified`, `unknown`, `reserved`) rather than implying certainty. If a decision is needed, it's a question for the user, not an assumption.

## Structure

4. **Per-epic prefixes.** Each epic uses a distinct prefix (`D-`, `B-`, `A-`, `W-`…). Detect collisions with existing issues in the repo before choosing; propose at the DAG gate.
5. **Waves → dependencies → critical path → priority.** Decompose into waves (the
   sequencing ordering); group issues into phases (the epic's phase column, e.g.
   scaffold/map/basecamp/sync); record dependencies; derive the critical path; assign
   priority (P0 = critical-path foundation, P1 = high, P2 = medium, P3 = polish).
6. **Epic body** carries: goal, context, the full spec (normative, inline), phases/tasks table, critical path, cross-repo readiness gate, out of scope, acceptance criteria.
7. **Issue body** carries: goal, context, the relevant spec detail (inline), out of scope, files to modify, checkbox acceptance criteria, dependencies, implementation notes, test plan.

## State & truth

8. **Completion = forge issue state**, never the epic task table. The updater reads open/closed/merged via the forge: GitHub `stateReason`; GitLab has no close-reason field — infer from labels/closed-by, ambiguous → ask. Plus linked PR/commit best-effort.
9. **Annotation pass (creation)**: after creating + linking, every body is updated in place — prefix references, `{SIBLING_EPIC}`, and `#{epic-number}` placeholders are replaced with the actual issue numbers + hyperlinks. The prefix → number registry (state file) drives this pass.
10. **Cross-repo dependencies** use native `blocked_by` links (globally-unique database ids) with markdown-hyperlink fallback; sibling epics reference each other by number.

## Quality

11. **Test plans are mandatory** on every issue; security/testing are part of every task, not a separate phase.
12. **Acceptance criteria are checkboxes** — testable, measurable, unambiguous.
13. **No over-engineering.** Each issue addresses a root cause; deliberate workarounds are recorded with their rationale.
