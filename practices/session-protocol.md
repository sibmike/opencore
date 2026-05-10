# Session Protocol

Covers what an agent does at session start, during active work, and at session end. Applies to every session regardless of task size.

---

## Entry-Point File Design

The root agent-configuration file is loaded into every session. Its role is to serve as an index to the knowledge base — not the knowledge base itself. It tells the agent what to read and in what order, then delegates specifics to domain docs.

Keep this file lean (roughly 100–150 lines). It contains:

- Session-start reading list (mandatory) — all behavior-defining docs that must be read on every new session
- Key inviolable rules — branching discipline, documentation trail requirements, and similar non-negotiables
- Project overview — 2–3 sentences
- Current build status — test counts, coverage
- Milestone status — what is done, what is next
- Environment pointer — links to a separate dev-environment doc; do not inline paths or commands
- Conventions summary — the 20% of rules that cover 80% of tasks, with a pointer to the full conventions doc
- Documentation map — one table linking every domain doc with a one-line description

Organize the highest-priority non-negotiable rules as a small numbered set near the top. These rules are always enforced regardless of what the session is doing.

Environment-specific paths, shell commands, and known pitfalls belong in a dedicated dev-environment doc — not in the entry-point file. Changes to environment details then require editing only one canonical location.

> See also: [documentation/doc-maintenance.md](../documentation/doc-maintenance.md)

---

## Session Start: Required Reads

Before doing any work in a new session, read the project's behavior-defining documents in this order:

1. The project entry point (overview, build status, doc map)
2. Working rules / agent behavior — task management, verification, feature lifecycle
3. Major change process — doc architecture, feedback loops
4. Git workflow — merge procedure, doc maintenance cadence
5. Conventions — code style, naming, plan-mode review prompt
6. Engineering values and judgment heuristics
7. Error log — mistakes to never repeat
8. Machine-specific memory — environment pitfalls
   > Worktree sessions may create a separate memory directory. The canonical memory is always at the configured path. If the worktree-specific directory is empty, read the canonical one.
9. If resuming work: read active execution plans

All listed files **must** be read before any implementation begins. Use the documentation map to find domain-specific docs as needed during the session — do not search the filesystem.

The entry-point file (item 1) is itself a numbered read — projects forking CORE may add a position 0 read of `docs/core-practices/CORE.md` before item 1, indexing the inherited rulebook before the project's own entry point.

> See also: [documentation/doc-maintenance.md](../documentation/doc-maintenance.md), [practices/feature-lifecycle.md](feature-lifecycle.md)

---

## Pre-Implementation Reads

Before writing any code for a major change, read these artifacts in order:

1. Execution plan (active plans directory) — find the current phase; read its tasks and acceptance criteria
2. Design decision (referenced in the execution plan header) — understand the architecture, schema, key queries, and boundary rules
3. Product spec (referenced in the execution plan header) — understand user stories, API surface, and resolution logic
4. Conventions doc — code patterns, naming, and testing patterns for this repository
5. Existing code (referenced in the execution plan phase tasks) — read files you will modify before editing them

The execution plan header links to all upstream artifacts. Follow those links — do not search for context independently.

> See also: [practices/feature-lifecycle.md](feature-lifecycle.md), [practices/plan-mode.md](plan-mode.md)

---

## Subagent Strategy

Use subagents liberally to keep the main context window clean.

- Offload research, exploration, and parallel analysis to subagents. Do not pollute the main context with search results.
- Assign one task per subagent. Do not combine unrelated searches.
- For complex problems, throw more compute at the problem via subagents rather than guessing.

---

## Task Management

Plan before coding.

- Write a plan with checkable items before writing any code.
- Check in with the user before starting implementation.
- Mark items complete as you go. Use a task tracker for multi-step tasks.
- Provide a high-level summary at each step.
- Add a review section when work is complete.
- Update behavior docs after corrections.

> See also: [practices/plan-mode.md](plan-mode.md)

---

## Cross-Session Continuity

Major changes often span multiple sessions. Never rely on conversation history for state — all decisions must be in versioned files.

To resume correctly:

- Read the proposals doc to understand what has been scoped.
- Read the design decision to understand architectural choices.
- Read the active execution plan to find the last completed phase.

Plan files serve as a session's authoritative implementation record. Name them descriptively to match the corresponding execution plan — not an auto-generated session title.

Rules for plan files:

- One plan file per feature. Append follow-up work rather than creating parallel files.
- Update in place on course corrections. Do not create a new file per pivot.
- Before closing a session, rewrite the plan as a historical record: for each planned section, document what actually happened versus what was speculated, including actual files, commits, test counts, and decisions.
- Add a "Final state" block (branch, commit hashes, test counts, deployment status) and a "Session reflection" block.

Do not create plan files for read-only research sessions, rejected plans (delete the file), or per-tool-use scratchpad work.

> See also: [practices/git-workflow.md](git-workflow.md), [documentation/doc-maintenance.md](../documentation/doc-maintenance.md)

---

## Machine-Specific Memory

Maintain a cross-session operational memory file outside the repository, in the agent's memory directory.

It should contain:

- Machine-specific environment paths (virtual environment, package manager, language runtime)
- Version control config (user, remote, branches)
- Common pitfalls and their fixes — things that waste time if forgotten
- Documentation structure quick-reference — where things live

It is **not** for encoding rules or conventions. Route those to project docs instead:

| Content type | Destination |
|---|---|
| Code conventions | Conventions doc |
| Debugging patterns | Debugging doc |
| Deployment steps | Deployment doc |
| Behavioral rules | Agent-behavior doc |

---

## Behavioral Contract Doc

Every project should maintain a behavioral contract doc — how the agent works, not what the code does.

It covers:

- Workflow orchestration — plan mode, subagent strategy, self-improvement loop
- Task management — plan first, verify, track, explain, capture lessons
- Verification rules — never mark work done without proving it works
- Feature development lifecycle
- Deployment planning — recursive resource checks, use up-to-date docs
- Core working principles — simplicity, no laziness, minimal impact

> See also: [practices/feature-lifecycle.md](feature-lifecycle.md), [practices/debugging.md](debugging.md)

---

## Self-Improvement Loop

After any correction from the user, immediately update behavior docs or the error log with the pattern. Write rules that prevent the same mistake from recurring. Iterate on those rules until the mistake rate reaches zero.

When an agent makes a mistake, ask: "What was missing from the harness?" — not "Why is the model bad?" Fix the environment (docs, lints, tests, context), not the symptom.

Human judgment and decisions should be encoded into persistent artifacts. Do not rely on remembering to tell the agent the same thing next session. Capture judgment once; enforce it continuously.

> See also: [practices/anti-patterns.md](anti-patterns.md)

---

## Session End: Reflection and Closure

Before ending every session, answer these questions:

1. Was the structured debugging protocol followed? If not, what was skipped and why?
2. Were there any moments of making changes without verified understanding of the root cause?
3. Was any behavioral or process rule violated? If so, does the agent-behavior doc need a new rule?
4. Is there a pattern here that will recur? If so, write a rule to prevent it.
5. Are all active execution plans still accurate? Update or archive stale ones.
6. Is the quality-score tracker still accurate? If grades changed, update them.
7. Is the session plan file aligned with the actual outcome? It must be rewritten as an execution record — not speculation — with a Final state block and a Session reflection block.

At session end, also ask: "Did I discover a new machine-specific pitfall or operational fact?"

Things that belong in personal memory (machine-local, follows the user across projects):

- Shell or CLI command quirks on this specific machine
- Environment path discoveries
- Infrastructure or account-specific gotchas
- Deployment timing or ordering dependencies unique to this environment
- Session workflow lessons that save time in future sessions

> See also: [documentation/checklist-protocol.md](../documentation/checklist-protocol.md), [practices/debugging.md](debugging.md)

---

<!-- Assembled from: agent-behavior.md §2. Subagent Strategy, agent-behavior.md §3. Self-Improvement Loop, agent-behavior.md §7. Task Management, agent-behavior.md §12. Session Start Checklist, CLAUDE.md §Agent Entry Point (preamble), CLAUDE.md §Session Start (MANDATORY), CLAUDE.md §Three Commandments (heading), CLAUDE.md §Three Commandments/1. Read behavior-defining files at session start, CLAUDE.md §Environment & Commands, core-beliefs.md §Process belief 12 (Agent failures are harness bugs), core-beliefs.md §Process belief 13 (Capture judgment once enforce continuously), harness-guidelines.md §3.4 Cross-Session Continuity, harness-guidelines.md §3.5 Pre-Implementation Reading Order, harness-guidelines.md §4.4 CLAUDE.md Role, harness-guidelines.md §4.6 MEMORY.md Role, harness-guidelines.md §4.7 Agent Behavior Doc Role, post-deploy-checklist.md §6. Memory Check, post-deploy-checklist.md §7. Session Reflection, workflows.md §Plan File Management -->
