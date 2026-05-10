# Project–CORE Divergence Review

A project's `docs/core-practices/` is forked from upstream CORE at a specific version. The fork is expected to drift — practices get extended, removed, or reordered as the project hits real conditions the inherited rule did not anticipate. The divergence review is how that drift gets measured, classified, and turned into upstream CORE PRs.

## When to run

Monthly, or at end-of-project, whichever comes first. End-of-project is the highest-yield moment because the full arc of drift is visible.

For long-running projects, the monthly cadence prevents drift from compounding silently. Reviewers comparing two months of drift can hold the context; reviewers comparing six months cannot.

## Output

Two artifacts:

1. **CORE PR** drafts — one per generalizable evolution. Each PR follows the format in [`core-update-gate.md`](core-update-gate.md).
2. **Divergence report** — a full diff with classifications for every drifted file. Lives at `<project>/docs/deltas/divergence-<YYYY-MM>.md` (or `divergence-final.md` for end-of-project reviews).

## Read first (mandatory inputs)

- Upstream CORE: `core/practices/`, `core/documentation/`, `core/evolution/`. The current canonical version.
- The project fork: `<project>/docs/core-practices/`. The drifted version.
- The project's lineage record: `<project>/docs/LINEAGE.md` (or equivalent) showing which CORE version the project forked from.
- Delta reports: `<project>/docs/deltas/`. The history of why drifts happened.

## The prompt (run verbatim)

<!-- Source: dreaming_system_v2.md lines 352-408 -->

```markdown
# Core Divergence Review

You are comparing a project's forked core-practices against the current
GIT CORE to identify generalizable evolution candidates.

## Inputs
- GIT CORE: core/practices/ and core/documentation/ (current canonical)
- PROJECT fork: <project>/docs/core-practices/ (drifted version)
- Delta reports: <project>/docs/deltas/ (history of why drifts happened)

## Process

### 1. Diff
For each file in docs/core-practices/, diff against the corresponding
CORE file. Categorize each divergence:

| File | Divergence | Category | Evidence |
|------|-----------|----------|----------|
| debugging.md | Added step for async service debugging | Generalizable? | 3 dreams, 2 ERRATA |
| session-protocol.md | Removed pre-session doc review for hotfix sessions | Project-specific | Only relevant when deploy cadence is daily |
| infra-management.md | Added "verify IAM before verify resource" ordering | Generalizable | ERR-003, ERR-008, 5+ sessions |

### 2. Classify
- **Project-specific drift:** Stays in project. Not promoted.
  (e.g., "check Cognito config" — only relevant if you use Cognito)
- **Generalizable evolution:** Would improve any project's core practices.
  (e.g., "verify IAM before verifying the resource it protects" — universal)
- **Contested:** Drift that might generalize but evidence is thin.
  Hold for data from other projects.

### 3. Draft CORE PR
For each generalizable evolution:

\`\`\`
FILE: core/practices/<file>
CURRENT: <what CORE says>
PROPOSED: <what the drift taught us>
SOURCE PROJECT: <project name>
EVIDENCE: <which deltas/dreams/ERRATA support this>
SESSIONS: <count of sessions that demonstrated this>
\`\`\`

### 4. PR Requirements
- ≥3 engineer approvals required
- Reviewers should include engineers from DIFFERENT projects
  (to verify the lesson generalizes beyond the source project)
- Each reviewer answers: "Would this have helped MY project?"
  If <2 say yes → hold, gather more cross-project evidence

## Output
- CORE PR (ready for multi-engineer review)
- Divergence report (full diff with classifications)
- Items held for cross-project validation
```

## Classification heuristics

When a divergence sits on the boundary between project-specific and generalizable, apply these tests:

- **The "different stack tomorrow" test.** If the project tore down its stack and rebuilt on a totally different runtime, would the rule still apply? Yes → generalizable. No → project-specific.
- **The "service name swap" test.** If you find-and-replace every service or vendor name in the rule with a generic placeholder, does the rule still read as a coherent practice? Yes → generalizable. No → project-specific.
- **The "would another project benefit" test.** Pick three project archetypes you have not worked on (a CLI tool, a mobile app, a data pipeline). For at least two of them, would this rule have caught a real failure mode? Yes → generalizable. No → project-specific.

A drift can be partially generalizable: the principle is universal but the example is specific. In that case, the CORE PR proposes the generalized principle alone; the example stays in the project fork.

## Contested handling

A contested drift is one where the evidence is thin or the lesson is plausible-but-unproven. Do not promote. Instead:

- Add the drift to the divergence report's Contested section.
- Tag it with the project archetype it came from.
- Wait for a second project to surface the same drift independently. Two-project convergence promotes a contested item to a CORE PR candidate.

A contested item that does not converge after three review cycles gets dropped, with a one-line note in the divergence report explaining why.

## Cadence interaction with delta extraction

The divergence review runs on a slower cadence than delta extraction. A typical schedule:

- Per session: dream.
- Every five dreams or weekly: delta extraction.
- Monthly or end-of-project: divergence review.

Delta extraction surfaces in-project signal. Divergence review surfaces upstream signal. The two are different jobs against different audiences and should not be collapsed.

<!-- Source: dreaming_system_v2.md (Section "Core Divergence Review"), authored 2026-05-10 -->
