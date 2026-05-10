# Dreaming

A dream is a structured reflection at session end. It reconstructs the gap between plan and reality, classifies what changed, and stages updates as PRs against the project, the project's forked core practices, the user preference file, and the ERRATA log. Dreams never mutate docs directly; they propose.

## When to run

Run a dream at the end of every session, after the post-deploy checklist's Net Infra Footprint Check passes. A session that touched no code still gets a dream — debugging-only sessions are where the highest-signal anti-pattern dreams come from.

If a session is interrupted mid-task and resumed later, the dream lives at the end of the resumed session, not the interrupted one. Reconstruct the full plan-vs-reality arc across both halves.

## Output

Write the dream to `<project>/docs/dreams/dream-<YYYY-MM-DD>.md`. If a dream already exists for today, suffix `-2`, `-3`, etc. Use the template at [`../templates/dream-report.md.template`](../templates/dream-report.md.template).

## Read first (mandatory inputs)

Before writing the dream, read:

- The project's forked core practices in `<project>/docs/core-practices/`. The dream classifies drift against these — if you do not know what the inherited rule says, you cannot detect drift.
- `<project>/ERRATA.md`. Mistakes already named.
- The last three dreams in `<project>/docs/dreams/`. Avoid re-proposing items already staged.

## The prompt (run verbatim)

<!-- Source: dreaming_system_v2.md lines 224-312 -->

```markdown
# Dream: Session Reflection & Knowledge Extraction

You are performing a dream — a structured reflection on the session
to extract learnings and stage documentation updates as PRs.

## Read First
- Project's forked core practices: <project>/docs/core-practices/
- ERRATA.md — known mistakes
- Recent dreams: <project>/docs/dreams/ (last 3)

## Phase 1: Delta

Reconstruct what happened. Be concrete.

- **Goal:** What was the session trying to accomplish?
- **Plan:** What was the approach?
- **Reality:** What actually happened?
- **Delta:** Where did plan ≠ reality? Name the gap. Name WHY.

## Phase 2: Assumption Autopsy

- Which assumptions held?
- Which broke? (Name each. Name what broke it.)
- Which new untested assumptions were introduced?

## Phase 3: Classify

For each significant delta:

| Delta | Project-specific? | Core-practice drift? | User preference? |
|-------|-------------------|---------------------|-----------------|
| ... | yes/no + reasoning | yes/no + which practice file | yes/no |

- **Project-specific:** Only matters in this codebase → stage PROJECT PR
- **Core-practice drift:** The forked practice file should be updated because
  reality revealed the inherited practice was incomplete, wrong, or missing
  a case → update docs/core-practices/<file> via PROJECT PR
  (this drift is measured at end-of-project against GIT CORE)
- **User preference:** A pattern about HOW this human works → stage USER PR

## Phase 4: Stage PRs

Do NOT apply changes directly. Produce staged proposals.

### PROJECT PR (reviewed by project engineers)
\`\`\`
FILE: <path>
ACTION: add | edit | delete
CHANGE: <specific diff>
REASONING: <why>
\`\`\`

### CORE PRACTICE DRIFT (applied to project's forked copy, measured later)
\`\`\`
FILE: docs/core-practices/<file>
INHERITED: <what CORE said>
PROPOSED: <what project reality demands>
REASONING: <what encounter revealed this>
\`\`\`

### ERRATA (if applicable)
\`\`\`
ERR-NNN: <title>
SEVERITY: high | medium | low
WHAT: <one paragraph>
ROOT CAUSE: <one sentence>
LESSON: <one sentence, generalized>
\`\`\`

### USER PR (reviewed by human)
\`\`\`
PREFERENCE: <what was observed>
EVIDENCE: <which session behaviors suggest this>
PROPOSED ADDITION: <specific line to add to user-preferences.md>
\`\`\`

## Phase 5: Meta

- Did the session follow debugging protocol? Where did it deviate?
- Were there "shotgun debugging" moments?
- Did any core practice get violated? Should the practice be updated or should I follow it better?
- Documentation debt: increased, decreased, or stable?

## Output

Write to: `<project>/docs/dreams/dream-<YYYY-MM-DD>[-N].md`
```

## What a good dream looks like

- **Concrete.** "The plan said deploy at 14:00; the deploy ran at 16:30 because the IAM check at step 3 failed and the recovery cost two hours." Not "the deploy took longer than expected."
- **Honest about assumption breaks.** Name the assumption. Name what broke it. If you do not know what broke it, say so.
- **Drift-aware.** When a practice was violated, ask: was the practice wrong, or did you skip it? Both answers are valid PRs — one updates the practice, one updates the human.
- **Bounded.** A dream is not a retrospective essay. Five concrete deltas beat fifteen vague ones.

## What a bad dream looks like

- Narrates the session without naming a single delta.
- Proposes vague PRs ("be more careful with infra"). PRs must be specific edits to specific files.
- Fabricates assumptions to fill the autopsy section. Only list assumptions that materially shaped the work.
- Promotes lessons to CORE drift when they are project-specific. CORE drift requires the rule to be plausibly universal — "verify IAM before the resource it protects" is universal; "check the audit-log table after writing to it" is project-specific.

## What happens to a dream

Dreams accumulate in `<project>/docs/dreams/`. Every five dreams (or weekly, whichever comes first) a delta extraction batch-processes them — see [`delta-extraction.md`](delta-extraction.md). Dreams themselves are never edited after writing; they are append-only session records.

<!-- Source: dreaming_system_v2.md (Section "Dream Prompt"), authored 2026-05-10 -->
