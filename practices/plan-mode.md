# Plan Mode

## When Plan Mode Is Mandatory

Enter plan mode before starting any task that involves 3 or more steps or any architectural decision. The threshold is low on purpose — ambiguity caught in planning is cheaper than ambiguity discovered mid-implementation.

Infra changes (new storage resources, IAM additions, runtime config changes, new services) always require plan mode before the first line of code, regardless of step count.

> See also: [practices/infra-management.md](infra-management.md)

## What Makes a Plan Adequate

A plan is adequate when it specifies:

- What will change and why.
- The sequence of steps in order.
- Verification steps — not just build steps.
- Any assumptions that, if wrong, would invalidate the approach.

Vague requirements produce vague implementations. Write enough detail upfront to make the approach falsifiable before a single line of code is written.

> See also: [practices/feature-lifecycle.md](feature-lifecycle.md)

## Approval Gates

Submit the plan for explicit approval before implementing. Approval is not implied by silence or by the absence of objection. Ask explicitly whether the user agrees before proceeding.

The same gate applies to each option presented during review: state the recommended option, explain the reasoning, and ask whether the user agrees or wants a different direction.

## Re-entering Plan Mode Mid-Implementation

Stop and re-plan immediately when any of these occur:

- The approach is not working and a fix feels like a workaround.
- A new fact emerges that invalidates an assumption in the original plan.
- The scope has grown beyond what the original plan described.

Do not push through. The question to ask: "Knowing everything I know now, what is the right approach?" Answer that question before writing more code.

> See also: [practices/anti-patterns.md](anti-patterns.md)

## Review Depth: Ask Before Starting

Before beginning any structured plan review, ask the user to choose a depth mode:

- **BIG CHANGE** — interactive, section-by-section, up to 4 top issues per section.
- **SMALL CHANGE** — interactive, one question per section.

Do not assume a default. Always ask first.

## Review Sections

Work through the four sections in order. Pause after each section for user feedback before continuing.

1. **Architecture** — Overall system design and component boundaries; dependency graph and coupling; data flow patterns and bottlenecks; scaling characteristics and single points of failure; security boundaries (auth, data access, API surfaces).

2. **Code quality** — Code organization and module structure; DRY violations; error handling patterns and missing edge cases; technical debt hotspots; over-engineered or under-engineered areas.

3. **Tests** — Coverage gaps (unit, integration, end-to-end); test quality and assertion strength; missing edge case coverage; untested failure modes and error paths.

4. **Performance** — Excessive query round-trips and data access patterns; memory-usage concerns; caching opportunities; slow or high-complexity code paths.

## Per-Issue Protocol

For every issue found during review:

1. Describe the problem concretely, with file and line references.
2. Present 2–3 options, including "do nothing" where reasonable.
3. For each option, specify: implementation effort, risk, impact on other code, and maintenance burden.
4. Give a recommended option and explain why, grounded in the project's canonical engineering values document rather than improvised principles.
5. Ask explicitly whether the user agrees or wants a different direction before proceeding.

Key values to encode in that canonical document and apply here: DRY, well-tested, right-sized engineering, explicit over clever, minimal blast radius, simplicity first.

> See also: [practices/session-protocol.md](session-protocol.md)

## Formatting Rules

- Number issues sequentially (1, 2, 3, …).
- Letter options per issue (A, B, C, …).
- The recommended option is always listed first (Option A).
- Each option label must include the issue number and letter — for example, "1A: Extract to utility" or "1B: Do nothing" — so the user is never confused about which issue an option addresses.
- Do not assume priorities on timeline or scale.
- After each review section, pause and ask for feedback before moving on.

<!-- Assembled from: agent-behavior.md §1. Workflow Orchestration, conventions.md §Plan Mode Review Prompt, conventions.md §Engineering Preferences, conventions.md §Before Starting: Ask the User, conventions.md §Review Sections, conventions.md §For Each Issue Found, conventions.md §Formatting Rules -->
