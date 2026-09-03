# Grug Principles — Complexity Is the Enemy

Universal engineering rules derived from "The Grug Brained Developer". Load this
file when working on implementation, architecture, planning, or review. Treat
every rule below as mandatory.

## Core

**Unnecessary complexity is the enemy.** Necessary complexity is a cost we pay
deliberately. The goal is not to write less code; it is to make the machine,
the code, the architecture, and the human operator understand no more than is
actually necessary. When in doubt, choose the simpler option.

## Decision Rules

1. **Say "no" to unnecessary complexity.** When a requirement genuinely
   requires complexity, accept that complexity deliberately rather than hiding
   or multiplying it. Do not rationalize rejecting legitimate requirements
   because "complexity bad."
2. **Build the smallest solution that satisfies the actual requirement.**
   When a full solution isn't justified, don't gold-plate. (Some work —
   auth, migrations, transactions, security, data integrity — is legitimately
   most of the code for a fraction of the visible functionality. That is
   necessary complexity, paid deliberately.)
3. **Don't factor early.** Wait for cut points to emerge from the code. A good
   cut point has a narrow interface hiding complexity internally. Bias toward
   waiting over premature abstraction.
4. **Prefer simple over clever.** Repeat code with small variation beats
   elaborate DRY abstractions. Put code on the thing that does the thing
   (locality over separation of concerns).
5. **Respect Chesterton's Fence.** Don't tear out code you don't understand.
   Understand the system first; respect code that works today even if
   imperfect.
6. **Keep refactors small** and never too far from shore. The system should
   work at every step.
7. **Don't optimize without a profile.** Real performance data first. Network
   calls are the usual culprit, not CPU.

## Engineering Practices

8. **Test at the lowest level that gives meaningful confidence.** Prefer
   integration tests where interactions between components are what you're
   trying to verify; use unit tests where they are the low-complexity
   solution. Keep a small, curated end-to-end suite working religiously.
   Write a regression test first when you find a bug. Mock rarely and
   coarsely.
9. **Make important behaviour observable.** Log meaningful state transitions,
   major branches, failures, request/job IDs, and external boundaries. Don't
   log noise.
10. **Prefer simple concurrency** — stateless handlers, independent job
    queues, optimistic concurrency.
11. **Use types where they reduce ambiguity and make code easier to use and
    refactor.** Don't introduce elaborate type machinery merely to satisfy
    theoretical correctness.
12. **Avoid generic abstractions until repeated concrete cases demonstrate
    that they simplify the code.**
13. **Design APIs around the common/simple case first.** Add escape hatches
    for uncommon cases rather than making every caller configure everything.
    Put common operations on the thing itself.
14. **Keep front ends simple** — plain HTML, minimal JS; be skeptical of big
    frameworks and fads.
15. **Value tools** — debugger, IDE completion, good logging infrastructure.

## Agent Rules

16. **Inspect before changing.** Understand the relevant code path, callers,
    tests, configuration, and constraints before proposing or implementing
    changes.
17. **Don't invent requirements.** Implement what is required, not what you
    imagine might eventually be required.
18. **Don't invent architecture.** Prefer existing project conventions and
    mechanisms over introducing new ones.
19. **Prefer deletion.** Before adding code, ask whether an existing
    mechanism, simplification, or deletion solves the problem.
20. **Don't solve hypothetical problems.** Don't add extension points,
    configuration, abstractions, validation, compatibility layers, or
    infrastructure for hypothetical future requirements. Future requirements
    can justify future code.
21. **Make changes coherent and reviewable.** Avoid bundling unrelated
    cleanup, refactoring, formatting, or modernization into a change.
22. **Verify your work.** After changing code, run the narrowest meaningful
    tests or checks. Expand verification when the change warrants it.
23. **Don't hide uncertainty.** If the correct behaviour is unclear,
    investigate or ask rather than inventing an assumption.
24. **Leave the code no more complex than the requirement demands.** If a
    change necessarily adds complexity, be able to explain why that complexity
    is required.
25. **Use what already exists.** Before introducing a new dependency,
    service, abstraction, utility, configuration mechanism, or framework,
    look for an existing project mechanism that already solves the problem.
    New machinery must earn its complexity.

## Mindset Rules

26. **Admit when it's too complex** (no Fear Of Looking Dumb). This empowers
    others to do the same.
27. **Accept impostor syndrome** — everyone feels it.
28. **Stay calm** — don't reach for the club; steer others with working
    demos.

## One-Line Summary

Unnecessary complexity very, very bad. Necessary complexity is a cost we pay
deliberately.
