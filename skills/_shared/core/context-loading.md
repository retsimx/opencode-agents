# Dynamic Context Loading Guide

Agents should not read all resources at once. Instead, load only necessary resources based on task type.
This saves context window and prevents confusion from irrelevant information.

**Path convention (host project root):** Agents run with cwd = the host project that contains `.agents/`. Every load path below is from that root.

| Kind | Format | Resolution Priority |
|------|--------|---------------------|
| Project Checklists (Tier 1) | `docs/checklists/<domain>.md` | Primary (checked first in host project root) |
| Shared | `.agents/skills/_shared/{core\|conditional\|runtime}/file.md` | Common framework protocols |
| Current skill resource | `.agents/skills/<skill-name>/resources/file.md` | Skill-specific fallback |
| Results & Post-Mortems (Tier 2) | `.agents/results/...` (e.g. `.agents/results/bugs/`) | Session artifacts & RCA reports |

Short names in the agent mapping tables mean files under that agent's `.agents/skills/<skill>/resources/` directory.

### Checklist Path Resolution Rule (CRITICAL)
When loading checklists (pre-flight or verification), agents and orchestrators MUST check `docs/checklists/<domain>.md` in the host project root first. If `docs/checklists/<domain>.md` does not exist, fall back to `.agents/skills/<skill>/resources/checklist.md`.

---

## Loading Order (Common to All Agents)

### Always Load (Required)
1. Skill `SKILL.md`: Auto-loaded by the runtime
2. Execution protocol: `.agents/skills/<skill>/resources/execution-protocol.md` when present; otherwise `.agents/skills/_shared/runtime/execution-protocol.md`

### Load at Task Start (Pre-Flight Tier 0 Injection)
3. **Pre-flight Tier 0 Checklist**: Load `docs/checklists/<domain>.md` from the host project root (fallback to `.agents/skills/<skill>/resources/checklist.md`). Tier 0 checklists are injected specifically for **Implementation Agents (`backend`, `frontend`, `db`) and Debug Agent** before starting implementation, while Planning Agent (`plan`) loads macro architectural context (`ARCHITECTURE.md`, `docs/design-docs/`, `docs/product-specs/`, `docs/plans/work/tech-debt-tracker.md`).
4. `.agents/skills/_shared/core/difficulty-guide.md`: Difficulty assessment (Step 0)

### Load Based on Difficulty
5. **Simple**: Proceed to implementation without additional loading
6. **Medium**: skill-local `examples.md` under `.agents/skills/<skill>/resources/`
7. **Complex**: skill-local `examples.md` + stack docs under that skill's `resources/` (e.g. `tech-stack.md`, `snippets.md`)

### Load During Execution as Needed
8. Domain checklist: Load `docs/checklists/<domain>.md` (fallback: skill-local `checklist.md`) at Step 4 (Verify)
9. skill-local `error-playbook.md`: Load only when errors occur
10. `.agents/skills/_shared/core/common-checklist.md`: For final verification of Complex tasks
11. `.agents/skills/_shared/runtime/coordination-protocol.md`: coordination for multi-agent sessions

### Load on Measurement / Exploration (Conditional)
12. `.agents/skills/_shared/conditional/quality-score.md`: Load when Quality Score measurement is needed (VERIFY/SHIP gates)
13. `.agents/skills/_shared/conditional/experiment-ledger.md`: Load when recording experiment results (after implementation changes)
14. `.agents/skills/_shared/conditional/exploration-loop.md`: Load only when a gate fails twice on the same issue

---

## Task Type → Resource Mapping by Agent

Unless noted, filenames below are under that agent's `.agents/skills/<skill>/resources/`.

**Checklist Resolution Rule**: For all domains, load `docs/checklists/<domain>.md` from the host project root first as the primary Tier 1 checklist/guardrail; fall back to skill-local `checklist.md` only if the host checklist does not exist.

### Backend Agent

| Task Type                     | Required Resources                          |
| ----------------------------- | ------------------------------------------- |
| Pre-flight / All tasks        | `docs/checklists/backend.md` (fallback: `checklist.md`) |
| CRUD API creation             | snippets.md (route, schema, model, test)    |
| Authentication implementation | snippets.md (JWT, password) + tech-stack.md |
| DB migration                  | snippets.md (migration)                     |
| Performance optimization      | examples.md (N+1 example)                   |
| Existing code modification    | examples.md + grep/glob/read                     |

### Frontend Agent

| Task Type           | Required Resources                                     |
| ------------------- | ------------------------------------------------------ |
| Pre-flight / All tasks | `docs/checklists/frontend.md` (fallback: `checklist.md`) |
| Component creation  | snippets.md (component, test) + component-template.tsx |
| Form implementation | snippets.md (form + Zod)                               |
| API integration     | snippets.md (TanStack Query)                           |
| Styling             | tailwind-rules.md                                      |
| Page layout         | snippets.md (grid) + examples.md                       |

### Mobile Agent

| Task Type        | Required Resources                                    |
| ---------------- | ----------------------------------------------------- |
| Pre-flight / All tasks | `docs/checklists/mobile.md` (fallback: `checklist.md`) |
| Screen creation  | snippets.md (screen, provider) + screen-template.dart |
| API integration  | snippets.md (repository, Dio)                         |
| Navigation       | snippets.md (GoRouter)                                |
| Offline features | examples.md (offline example)                         |
| State management | snippets.md (Riverpod)                                |

### Debug Agent

| Task Type       | Required Resources                                                |
| --------------- | ----------------------------------------------------------------- |
| Pre-flight / All bugs | Relevant `docs/checklists/<domain>.md` + `.agents/results/bugs/` post-mortems |
| Frontend bug    | common-patterns.md (Frontend section)                             |
| Backend bug     | common-patterns.md (Backend section)                              |
| Mobile bug      | common-patterns.md (Mobile section)                               |
| Performance bug | common-patterns.md (Performance section) + debugging-checklist.md |
| Security bug    | common-patterns.md (Security section)                             |

### QA / Review Agent

| Task Type            | Required Resources                                  |
| -------------------- | --------------------------------------------------- |
| Pre-flight / All reviews | `docs/checklists/common.md` + `docs/checklists/<domain_being_reviewed>.md` (e.g. `backend.md`, `frontend.md`, `db.md`) (fallback: `checklist.md`) |
| Domain Review (Backend / Frontend / DB) | `docs/checklists/common.md` + `docs/checklists/<domain_being_reviewed>.md` (fallback: `checklist.md`) |
| Security / Performance / Accessibility review | `docs/checklists/common.md` + relevant domain checklist (`docs/checklists/<domain_being_reviewed>.md`) |
| Full audit           | `docs/checklists/common.md` + `docs/checklists/` (all domain checklists) + `checklist.md` (full) + self-check.md |
| Quality scoring      | `.agents/skills/_shared/conditional/quality-score.md` (measurement protocol via Bash)    |

### Architecture Agent

| Task Type                    | Required Resources                                                         |
| ---------------------------- | -------------------------------------------------------------------------- |
| Architecture recommendation  | methodology-selection.md + output-templates.md                             |
| Design review                | methodology-selection.md + checklist.md + stakeholder-synthesis.md         |
| Design-twice comparison      | methodology-selection.md + stakeholder-synthesis.md + examples.md          |
| ATAM-style analysis          | methodology-selection.md + stakeholder-synthesis.md + output-templates.md  |
| CBAM-style prioritization    | methodology-selection.md + output-templates.md                             |
| ADR generation               | output-templates.md                                                        |

### Developer Workflow Expert

| Task Type                   | Required Resources                                            |
| --------------------------- | ------------------------------------------------------------- |
| API Workflow Setup          | api-workflows.md + validation-pipeline.md |
| Database Migration Workflow | database-patterns.md                                |
| Release Coordination        | release-coordination.md                             |
| Troubleshooting             | troubleshooting.md                                  |

### TF Infra Agent

| Task Type                   | Required Resources                                                       |
| --------------------------- | ------------------------------------------------------------------------ |
| Infrastructure Provisioning | multi-cloud-examples.md + policy-testing-examples.md |
| Cost Analysis               | cost-optimization.md                                           |

### PM Agent

| Task Type                 | Required Resources                                           |
| ------------------------- | ------------------------------------------------------------ |
| New project planning      | examples.md + task-template.json + `.agents/skills/_shared/core/api-contracts/template.md` + `ARCHITECTURE.md` + `docs/design-docs/` + `docs/product-specs/` + `docs/plans/work/tech-debt-tracker.md` |
| Feature addition planning | examples.md + grep/glob/read (understand existing structure) + `ARCHITECTURE.md` + `docs/design-docs/` + `docs/product-specs/` + `docs/plans/work/tech-debt-tracker.md`    |
| Refactoring planning      | grep/glob/read only + `ARCHITECTURE.md` + `docs/design-docs/` + `docs/plans/work/tech-debt-tracker.md`                          |

### Design Agent

| Task Type                   | Required Resources                                                       |
| --------------------------- | ------------------------------------------------------------------------ |
| Design system creation      | reference/typography.md + reference/color-and-contrast.md + reference/spatial-design.md + design-md-spec.md |
| Landing page design         | reference/component-patterns.md + reference/motion-design.md + prompt-enhancement.md + examples/landing-page-prompt.md |
| Design audit                | checklist.md + anti-patterns.md                                          |
| Design token export         | design-tokens.md                                                         |
| Stitch MCP integration      | stitch-integration.md                                                    |
| 3D / shader effects         | reference/shader-and-3d.md + reference/motion-design.md                  |
| Accessibility review        | reference/accessibility.md + checklist.md                                |

---

## Orchestrator Only: Composing Subagent Prompts

When the Orchestrator composes subagent prompts, reference the mapping above
to include only resource paths matching the task type in the prompt.

Additionally, under the **Universal File-First State I/O Architecture**, the orchestrator MUST explicitly inject context variables and output mandates into every subagent prompt template:
- `SESSION_ID`: Resolved session ID (`Issue Slug` -> `Conversation Prefix` -> `YYYYMMDD-HHMMSS`).
- `TASK_SLUG`: Kebab-case identifier of the assigned task (e.g., `cart-api`, `auth-jwt`).
- `OUTPUT_FILE`: Designated repository artifact path (`.agents/results/{type}-{role}-{taskSlug}-{sessionId}[-{index}].md`).
- `UPSTREAM_ARTIFACTS`: Explicit file paths to upstream subagent outputs (Pass-by-Reference for Zero-Context Relay).

```
Prompt composition:
1. Context Variables & Output Mandate:
   - SESSION_ID: <resolved-session-id>
   - TASK_SLUG: <task-slug>
   - OUTPUT_FILE: .agents/results/{type}-{role}-{taskSlug}-{sessionId}.md
   - UPSTREAM_ARTIFACTS: [paths to upstream deliverable files, if applicable]
   - Mandate: Write exhaustive deliverables to OUTPUT_FILE, verify write, return standard 4-line chat summary.
2. Agent SKILL.md's Core Rules section
3. Pre-flight Tier 0 Checklist Injection: docs/checklists/<domain>.md in host root (fallback: .agents/skills/<skill>/resources/checklist.md)
4. Execution protocol (.agents/skills/<skill>/resources/execution-protocol.md, else .agents/skills/_shared/runtime/execution-protocol.md)
5. Resources matching task type (resolve under .agents/skills/<skill>/resources/)
6. error-playbook.md (always include; recovery is essential)
7. Coordination protocol: .agents/skills/_shared/runtime/coordination-protocol.md
```

This approach avoids loading unnecessary resources and prevents context pollution, maximizing subagent and orchestrator context efficiency.

---

## Conditional Protocol Loading (Measurement & Exploration)

The following protocols are **NOT** loaded at Phase 0 / Step 0. They are loaded on-demand:

| Protocol | Trigger | Loaded By |
|----------|---------|-----------|
| `.agents/skills/_shared/conditional/quality-score.md` | VERIFY or SHIP phase begins | Orchestrator (passes to QA agent prompt) |
| `.agents/skills/_shared/conditional/experiment-ledger.md` | First experiment recorded | Orchestrator (inline, after IMPL baseline) |
| `.agents/skills/_shared/conditional/exploration-loop.md` | Same gate fails twice on same issue | Orchestrator (inline, before spawning hypothesis agents) |

**Budget impact**: ~750 tokens total if all 3 loaded, but since loading is conditional, typical sessions load 1-2 only.
Flash-tier budget remains within ~3,100 token allocation for most sessions.
