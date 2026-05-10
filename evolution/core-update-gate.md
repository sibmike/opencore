# CORE Update Gate

CORE evolves slowly and on evidence from many projects. The gate exists because a rule that helped one project may actively harm another — promoting too eagerly poisons the shared rulebook.

## What CORE accepts

A CORE PR proposes a new practice doc, an edit to an existing practice, a new section, or a removal. Sources of CORE PRs:

- Project-core divergence reviews ([`project-core-divergence.md`](project-core-divergence.md)) that classify a drift as Generalizable.
- Cross-project ERRATA convergence — the same anti-pattern surfaces in two or more independent projects.
- Direct authorship by a CORE maintainer responding to repeated cross-project signal.

A CORE PR does not get accepted because one project found it useful. Useful-in-one-project belongs in that project's forked `docs/core-practices/`, not upstream.

## Approval threshold

- **Three engineer approvals minimum.**
- **Approvers come from at least two distinct projects.** Three approvers from the same project = the rule is project-specific dressed as universal. Hold.
- **Each approver answers the question:** "Would this rule have helped MY project?" If two or more answers are no, hold the PR and gather more cross-project evidence.

The "would this have helped" test is asked privately — approvers commit to a yes/no before reading each other's answers, to avoid groupthink.

## What blocks a CORE PR

- Names a specific technology, service, framework, language, or vendor. Generalize first; if generalization removes the meaning, the rule was project-specific.
- Cites only one project's evidence even after engineers from other projects reviewed.
- Proposes a removal that any project's `core-practices/` fork still actively relies on. Removals require the same threshold as additions plus explicit notice to fork holders.
- Makes a contradictory edit to an existing rule without resolving the contradiction. Two rules cannot disagree silently.

## CORE PR format

```
FILE: core/practices/<file>            # or core/documentation/<file>, core/evolution/<file>
CURRENT: <what CORE says today, verbatim>
PROPOSED: <what the new text says, verbatim>
SOURCE PROJECT: <project name where the lesson originated>
EVIDENCE:
  - Project A: <delta-batch ref, dream count, ERRATA refs>
  - Project B: <delta-batch ref, dream count, ERRATA refs>
SESSIONS DEMONSTRATING THIS: <count, summed across projects>
APPROVERS:
  - <engineer> (project <X>) — yes / no
  - <engineer> (project <Y>) — yes / no
  - <engineer> (project <Z>) — yes / no
WOULD-THIS-HAVE-HELPED RESULT: <yes count> / <no count>
```

## Versioning impact

CORE follows semver:

- **Patch (`v0.1.0` → `v0.1.1`):** Typo fixes, formatting, clarifications that do not change meaning.
- **Minor (`v0.1.0` → `v0.2.0`):** New sections within existing docs, new examples, deepened guidance, additive scaffolding (new evolution helpers, new templates).
- **Major (`v0.1.0` → `v1.0.0`):** New top-level docs, removed docs, contradictory updates that change meaning of an existing rule.

A CORE PR carries its proposed version bump in the PR description. Reviewers may downgrade or upgrade the bump as part of approval.

## Lineage tracking

Every accepted CORE PR appends an entry to `core/CORE-CHANGELOG.md` recording:

- New version number.
- Date.
- Source project(s).
- Approvers (with their project affiliations).
- One-paragraph summary of what changed and why.

The changelog is the audit trail. Forked projects compare their `docs/core-practices/` against upstream by walking the changelog forward from the version they forked.

## What auto-applies

Nothing. CORE updates do not auto-propagate to forked projects. Each project decides whether and when to pull a new CORE version into its fork. The fork timestamp lives in `<project>/docs/LINEAGE.md` (or equivalent); the project may stay on an older CORE indefinitely.

## Review gates summary

<!-- Source: dreaming_system_v2.md lines 414-422 -->

| What | Reviewed by | Approval threshold | Auto-apply? |
|------|------------|-------------------|-------------|
| PROJECT doc updates | Project engineers | Standard PR review | No |
| ERRATA entries | Project engineers | Standard PR review | No |
| Core-practice drift (project fork) | Project engineers | Standard PR review | No |
| USER preference updates | The human | Human reviews PR | No |
| GIT CORE evolution | ≥3 engineers from ≥2 projects | All approve | No |

Nothing auto-applies. Everything is a PR. The dream process generates proposals. Humans decide.

<!-- Source: dreaming_system_v2.md (Section "Review Gates Summary"), authored 2026-05-10 -->
