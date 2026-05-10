# Git Workflow

## Branching Model

Use a two-branch model: one stable, protected branch (e.g., `main`) and one active integration branch (e.g., `dev`).

- The integration branch is always ahead of or equal to the protected branch.
- All commits land on the integration branch first.
- Never commit directly to the protected branch — even hotfixes flow through the integration branch first.

For feature work, cut a short-lived `feature/<short-description>` branch off the integration branch. For bug fixes, use `fix/<short-description>`. Merge back to the integration branch when done; delete the short-lived branch after merge.

> See also: [practices/feature-lifecycle.md](feature-lifecycle.md)

## Commit Messages

Write commit messages in imperative mood. The summary line is concise and self-contained. Example pattern: `Add retry logic to token refresh` — not `Added retry logic` or `This commit adds retry logic`.

> See also: [documentation/writing-style.md](../documentation/writing-style.md)

## Merge Procedure (Integration → Protected)

1. Commit and push all work to the integration branch.
2. Check out the protected branch.
3. Merge the integration branch into the protected branch.
4. Push the protected branch.
5. Verify both branches resolve to the same commit.

Never skip the integration branch, even under time pressure. A direct commit to the protected branch bypasses review, breaks the invariant, and requires a corrective merge that may introduce conflicts.

> See also: [practices/plan-mode.md](plan-mode.md)

<!-- Assembled from: CLAUDE.md §Three Commandments / 2. Git commit discipline: dev → main, workflows.md §Git Workflow, workflows.md §Merge Procedure (dev → main) -->
