# Dynamic Context Loading Guide

Agents should not read all resources at once. Instead, load only necessary resources based on task type.
This saves context window and prevents confusion from irrelevant information.

**Path convention (host project root):** Agents run with cwd = the host project that contains `.agents/`. Every load path below is from that root.

| Kind | Format |
|------|--------|
| Shared | `.agents/skills/_shared/{core\|conditional\|runtime}/file.md` |
| Current skill resource | `.agents/skills/<skill-name>/resources/file.md` |
| Results | `.agents/results/...` |

Short names in the agent mapping tables mean files under that agent's `.agents/skills/<skill>/resources/` directory.

---

## Loading Order (Common to All Agents)

### Always Load (Required)
1. Skill `SKILL.md`: Auto-loaded by the runtime
2. Execution protocol: `.agents/skills/<skill>/resources/execution-protocol.md` when present; otherwise `.agents/skills/_shared/runtime/execution-protocol.md`

### Load at Task Start
3. `.agents/skills/_shared/core/difficulty-guide.md`: Difficulty assessment (Step 0)

### Load Based on Difficulty
4. **Simple**: Proceed to implementation without additional loading
5. **Medium**: skill-local `examples.md` under `.agents/skills/<skill>/resources/`
6. **Complex**: skill-local `examples.md` + stack docs under that skill's `resources/` (e.g. `tech-stack.md`, `snippets.md`)

### Load During Execution as Needed
7. skill-local `checklist.md`: Load at Step 4 (Verify)
8. skill-local `error-playbook.md`: Load only when errors occur
9. `.agents/skills/_shared/core/common-checklist.md`: For final verification of Complex tasks
10. `.agents/skills/_shared/runtime/coordination-protocol.md`: coordination for multi-agent sessions

### Load on Measurement / Exploration (Conditional)
11. `.agents/skills/_shared/conditional/quality-score.md`: Load when Quality Score measurement is needed (VERIFY/SHIP gates)
12. `.agents/skills/_shared/conditional/experiment-ledger.md`: Load when recording experiment results (after implementation changes)
13. `.agents/skills/_shared/conditional/exploration-loop.md`: Load only when a gate fails twice on the same issue

---

## Task Type → Resource Mapping by Agent

Unless noted, filenames below are under that agent's `.agents/skills/<skill>/resources/`.

### Backend Agent

| Task Type                     | Required Resources                          |
| ----------------------------- | ------------------------------------------- |
| CRUD API creation             | snippets.md (route, schema, model, test)    |
| Authentication implementation | snippets.md (JWT, password) + tech-stack.md |
| DB migration                  | snippets.md (migration)                     |
| Performance optimization      | examples.md (N+1 example)                   |
| Existing code modification    | examples.md + grep/glob/read                     |

### Frontend Agent

| Task Type           | Required Resources                                     |
| ------------------- | ------------------------------------------------------ |
| Component creation  | snippets.md (component, test) + component-template.tsx |
| Form implementation | snippets.md (form + Zod)                               |
| API integration     | snippets.md (TanStack Query)                           |
| Styling             | tailwind-rules.md                                      |
| Page layout         | snippets.md (grid) + examples.md                       |

### Mobile Agent

| Task Type        | Required Resources                                    |
| ---------------- | ----------------------------------------------------- |
| Screen creation  | snippets.md (screen, provider) + screen-template.dart |
| API integration  | snippets.md (repository, Dio)                         |
| Navigation       | snippets.md (GoRouter)                                |
| Offline features | examples.md (offline example)                         |
| State management | snippets.md (Riverpod)                                |

### Debug Agent

| Task Type       | Required Resources                                                |
| --------------- | ----------------------------------------------------------------- |
| Frontend bug    | common-patterns.md (Frontend section)                             |
| Backend bug     | common-patterns.md (Backend section)                              |
| Mobile bug      | common-patterns.md (Mobile section)                               |
| Performance bug | common-patterns.md (Performance section) + debugging-checklist.md |
| Security bug    | common-patterns.md (Security section)                             |

### QA Agent

| Task Type            | Required Resources                                  |
| -------------------- | --------------------------------------------------- |
| Security review      | checklist.md (Security section)                     |
| Performance review   | checklist.md (Performance section)                  |
| Accessibility review | checklist.md (Accessibility section)                |
| Full audit           | checklist.md (full) + self-check.md                 |
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
| New project planning      | examples.md + task-template.json + `.agents/skills/_shared/core/api-contracts/template.md` |
| Feature addition planning | examples.md + grep/glob/read (understand existing structure)     |
| Refactoring planning      | grep/glob/read only                                              |

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

```
Prompt composition:
1. Agent SKILL.md's Core Rules section
2. Execution protocol (.agents/skills/<skill>/resources/execution-protocol.md, else .agents/skills/_shared/runtime/execution-protocol.md)
3. Resources matching task type (resolve under .agents/skills/<skill>/resources/)
4. error-playbook.md (always include; recovery is essential)
5. Coordination protocol: .agents/skills/_shared/runtime/coordination-protocol.md
```

This approach avoids loading unnecessary resources, maximizing subagent context efficiency.

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
