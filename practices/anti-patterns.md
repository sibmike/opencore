# Anti-Patterns

Generalizable mistakes extracted from concrete incidents. Each section names the pattern, states the rule, and — where useful — gives an anonymized example of the failure mode.

> See also: [`practices/debugging.md`](debugging.md) for the full Observe→Hypothesize→Verify→Plan→Implement protocol.  
> See also: [`practices/infra-management.md`](infra-management.md) for infrastructure-specific discipline.  
> See also: [`practices/security-principles.md`](security-principles.md) for security-specific rules.

---

## Code quality and change discipline

**Simplicity first.** Make every change as simple as possible. Impact minimal code. Do not add features, refactor surrounding code, or make "improvements" beyond what was asked. A bug fix does not need surrounding code cleaned up. A simple feature does not need extra configurability.

**Minimal blast radius.** Changes touch only what is necessary. No temporary fixes — find root causes. Senior developer standards apply to every commit.

**Thoughtfulness over speed.** When in doubt, handle the error case. An unhandled error in production is worse than a slightly longer implementation.

**Elegance check on non-trivial changes.** For any non-trivial change, pause and ask: "Is there a more elegant way?" If a fix feels hacky, implement the elegant solution instead. Skip this for obvious one-line fixes — do not over-engineer a typo fix.

**Explicit over clever.** No metaprogramming, no dynamic imports, no magic that requires deep context to understand. If a reader needs to "just know" something to understand the code, that is a design failure — make the dependency or assumption explicit.

**Right-sized engineering.** Avoid premature abstraction and unnecessary complexity, but also avoid fragile hacks. Handle the edge cases you can foresee; do not build for edge cases you are imagining.

**Boring technology.** Choose composable, well-documented libraries over exotic alternatives. When an external library's behavior is opaque, consider reimplementing the needed subset with full test coverage.

**DRY as a core value.** When you see two implementations of the same logic, extract to a shared module. Centralize invariants in one place.

**Shell command safety.** Every shell command issued by an automated agent must be accompanied by an exact description of what it does and what it accomplishes. Confirm no unintended harm before execution.

---

## API and data layer design

**Validate at boundaries, not in the middle.** Parse and validate input at system entry points and at external service response boundaries. Interior code trusts that validated types are correct. Avoid redundant validation in business logic — it creates noise and false safety.

**Single authoritative schema per boundary.** Never duplicate validation logic across layers — trust the API contract and a single schema definition.

**Strict typing everywhere.** Use strict type-checking mode. Disallow implicit escape-hatch types. A type escape hatch in a statically typed language is a deferred runtime failure.

**Never raise transport-layer exceptions from domain logic.** Domain and service layers must not throw HTTP status errors. A central handler maps domain exceptions to transport responses.

**Async I/O throughout.** Never block an async event loop with synchronous I/O calls. Keep all I/O async end to end.

**Single data-fetching layer.** Designate one data-fetching library and use it exclusively. Never fetch server data outside that layer. Never store server-synchronized data in application global state.

**Normalize data shape at the boundary.** Case conversion, field renaming, and other shape normalization belong at the API boundary, not scattered through the application.

**Tier cache staleness by volatility.** Effectively-static data can cache indefinitely. Configuration-level data uses minute-scale staleness. Volatile operational data uses short or zero stale windows. For long-running async backend jobs, use exponential-backoff polling with a total timeout guard — do not use a fixed interval.

**Invalidate after mutation.** After every successful write, proactively invalidate all cached data that could be stale as a result. Do not rely on TTL-based expiry for correctness after writes.

**Storage technology matches data shape.** Content entities with flexible schemas belong in a document store. Relationship and permission data requiring joins belongs in a relational database. Do not over-engineer.

**Strict dependency direction.** Outer layers (request handlers) may depend on inner layers (services, repositories), but inner layers must never import from outer layers. Lateral dependencies between handlers of the same layer are also forbidden. Cross-cutting concerns enter through explicit dependency providers, not direct imports.

**Document store GSI key null guard.** If a secondary index key attribute would be null for a given item, omit the attribute entirely from the write request. Writing a null value for an index key attribute causes a validation error. Items without the attribute are simply excluded from the index, which is the correct behavior.

**URL identifiers must be immutable and unique by construction.** Any URL identifier resolving to a security-bearing entity — grants, payments, profile control, account state — must derive from a guaranteed-unique, account-scoped, immutable value such as the platform's internal account ID or a deterministic transform of it.

Profile attributes users can set themselves (display names, slugs, linked third-party IDs) are **not** identifiers. They are profile data subject to collisions and mutation. Even ignoring uniqueness, a mutable attribute used as a URL identifier becomes stale when the user updates their profile — breaking printed links, bookmarks, and shared URLs.

Secondary indexes do not enforce uniqueness. A secondary index on a non-unique attribute returns multiple rows. Code that takes the first item from such a query is a footgun whenever uniqueness is assumed. Iterate over results or change the identifier.

Validate "unique" assumptions against production data before shipping. A count query would surface collisions before the feature goes live. A salted hash of an immutable account identifier is a valid, stable, opaque URL form.

Single-purpose lookups belong in dedicated, isolated data stores — not as indexes or extra attributes on busy shared tables. A new isolated store beats an index on a shared table on every operational axis: easier to reason about, independently rollbackable, and zero blast radius on the shared table's lifecycle. The instinct to "minimize new resources" is not a virtue — minimize blast radius per change.

> See also: [`practices/security-principles.md`](security-principles.md)

**Document intended limits before implementing them.** Record intended rate limits in the architecture doc, organized by the risk each limit mitigates. Annotate unimplemented limits as tech debt so they are visible and trackable rather than silently missing.

---

## Infrastructure drift / works-locally-fails-in-prod

**If the code works locally but not in deployment, diagnose config before touching code.** The problem is almost certainly configuration, not code. Code changes are the last resort, not the first.

**Client-side environment variables inlined at build time must exist during the build step, not just at runtime.** Use the repository's environment file as the single source of truth. Do not set the same variable in both a hosting-console environment store and a committed environment file — the console value wins silently.

**SDK response key names differ across services from the same vendor.** Never assume casing or naming is consistent across services. Verify against the actual service model. Test mocks must match the real SDK response contract, not an assumed shape.

**Verify exact identifier formats before deploying to a new model or service tier.** Do not assume the identifier format from previous versions applies to new ones. Format variations — date suffixes, version tags, region prefixes — are common across generations and service tiers. Always check vendor documentation for the exact invocation identifier format.

**Never copy a request-body fragment from one API surface to another without re-verifying.** Different API surfaces from the same vendor accept different request shapes. Fragments are not portable. If a content block is missing a required field, the error message names the missing field — trust it.

**Account deletion and re-signup flows must handle all possible authentication states.** Orphaned auth-provider state after account deletion is a common source of re-signup failures.

**Worktrees do not include gitignored directories.** Any toolchain resolving dependencies from a relative path will fail inside a worktree. For package-manager-resolved toolchains, run tests against the main repo's installed dependencies. For interpreter-resolved toolchains, tests can run from the worktree if the interpreter is referenced by absolute path. Test files written in a worktree are only exercised against the worktree's source if the test runner is invoked from there — if the runner is hardcoded to the main repo, worktree test files are untestable until merged.

**Local-dev infrastructure triggers should be no-ops without real credentials.** Guard async job queues and external service calls with an empty/null config check so they become no-ops in local development. This prevents local runs from accidentally firing production side-effects.

**Every write endpoint must enforce its own authorization check.** When adding authenticated access to a previously restricted area, audit all routes in that area for read-vs-write separation. Frontend guards are defense-in-depth — backend enforcement is the source of truth.

> See also: [`practices/infra-management.md`](infra-management.md)

---

## Root-cause misdiagnosis

**CORS errors on non-OPTIONS requests are symptoms of backend failures, not CORS misconfiguration.** When the backend throws an unhandled exception, the error response may not include CORS headers. Always check backend logs before diagnosing CORS. The backend log shows the real error; the browser console shows the symptom.

**When a fix does not work after deploy, re-examine the root cause.** Do not iterate on the same hypothesis. The error message itself usually contains the clue. Do not compare with "working" code in the same codebase as a correctness check — it may have the same bug you are trying to diagnose.

**Any page using client-side navigation hooks that require async data must have a Suspense boundary.** Type checking alone does not catch this — only a build run reveals the failure. Static generation at build time cannot resolve async hooks.

**Larger or slower models require longer client timeouts.** Review timeout settings whenever switching to a model with different latency characteristics. API surface identifier formats also vary by model generation — verify against the model's documentation, not another model's known-good format.

**Mobile hit areas do not move with CSS visual transforms.** Using CSS transforms to toggle visibility of interactive panels on mobile causes the touch/click area to stay at the original position while the element appears elsewhere. Use `display:none` or equivalent to fully remove the element from layout and event handling.

<!-- CONTRADICTION: Fragment 20 (anti-pattern 4.4) says "if code works locally but fails in deployment, the problem is almost certainly configuration, not code." Fragment 26 (ERR-004) says "when a fix doesn't work after deploy, re-examine the root cause rather than iterating on the same hypothesis." These are compatible principles but they apply to different phases of the same scenario — the latter is about persistence after first diagnosis, not a contradiction. No flag needed. -->

---

## Premature action and shotgun debugging

**Never change working code based on assumptions about what a platform requires.** If the system was working before, the platform config was fine. The problem is elsewhere. Verify before changing.

**Verify fixes before deploying.** When possible, check config/infrastructure in the relevant console before pushing. For code changes, test locally first and reason about whether the change affects the deployment pipeline.

**Isolate problems.** Fix one thing at a time. Verify each fix independently. Never make a change that could affect a working system while debugging an unrelated issue.

**When a production incident is traced to a newly added pattern, restore first, then improve separately.** Fix with minimum change to restore working state. Isolate the follow-up — re-adding the more complex feature — as a separate verified step. Never bundle the safety restore with the improvement.

> See also: [`practices/debugging.md`](debugging.md)

---

## Scope creep and over-engineering

**Do not add what was not asked for.** A bug fix does not need surrounding code cleaned up. A simple feature does not need extra configurability. Adding unrequested work increases blast radius and introduces risk with no scoped benefit.

**Challenge your own work before presenting it.** Ask: "Is there a more elegant way?" A fix that feels hacky probably is. Implement the elegant solution rather than shipping the first one that passes tests.

**Do not couple a single-purpose lookup to a hot shared resource.** Minimize blast radius per change. An isolated new resource beats an index or extra column on a busy shared resource. Rollback simplicity is a real engineering property — deleting an isolated resource is the cleanest possible undo.

> See also: [`practices/architecture-principles.md`](architecture-principles.md)

---

## Constraint enforcement

**Encode constraints deterministically wherever possible.** Rules that depend on agents remembering to follow them will eventually be violated. Use linters, CI checks, schema validation, and structural tests. Linter error messages should teach — they double as context for agents.

Reserve LLM-based enforcement only for rules that cannot be expressed mechanically (for example, "is this commit message clear?"). When documentation falls short, promote the rule into code — a lint check that fires on every build is more reliable than a doc that might not be read.

**Three levels of constraint reliability:**

| Level          | Mechanism                                     | Reliability |
|----------------|-----------------------------------------------|-------------|
| Mechanical     | Linters, type checkers, CI gates, formatters  | Highest     |
| Structural     | Tests verifying architecture invariants       | High        |
| Agent-assisted | LLM self-checks during workflow               | Medium      |

Always prefer a higher-reliability mechanism. Promote agent-assisted constraints to mechanical ones when the pattern becomes clear.

**Minimum deterministic constraints for any project:**
- Code formatting (automated formatter) — eliminates style debates
- Import ordering (linter rule) — consistent dependency structure
- Type safety (type checker) — catches errors before runtime
- API schema validation (schema library) — contract enforcement
- Test gate (CI: tests must pass before merge) — prevents regressions
- Secret prevention (ignore-file patterns) — security baseline

**Structural constraints to build for any layered codebase:**
- Dependency direction lint: lower-layer modules cannot import from higher-layer modules
- File size limits: CI check preventing any file from exceeding a line threshold
- Naming conventions: automated lint on filenames and exports
- Schema boundary validation: test ensuring all API responses pass through the schema validation layer

**Constraints should fail loudly.** Error messages must explain why and how to fix — they are agent context. Prefer prevention (pre-commit) over detection (post-merge): catching errors early is cheaper. Every constraint must be documented — an undocumented rule is invisible to agents.

**When you find yourself giving the agent the same correction twice, add a constraint.** Capture human judgment once; enforce it continuously. The agent should write the harness fix itself — the human's job is to identify the gap and direct the fix.

**Agent failures are harness bugs, not just model limitations.** After every recurring issue, ask: "What doc, lint rule, or test would have prevented this?"

**Diagnostic rule for recurring failures:** Wrong output once? Likely a context problem — update docs. Slow degradation over weeks? That is a harness problem — add mechanical enforcement.

> See also: [`documentation/checklist-protocol.md`](../documentation/checklist-protocol.md)  
> See also: [`practices/session-protocol.md`](session-protocol.md)

---

## Doc-trail skips and error logging

**Read the project error log before any session involving high-risk file operations.** It documents severe, previously committed mistakes — with specific reproduction conditions — that must never be repeated. This pointer belongs as a highlighted critical warning at the top of the project entry-point document.

**Maintain a living error log.** At the end of every session, ask: "Did I make a severe mistake during this session?"

Qualifying criteria for an entry:
- Data loss or broken production state
- Three or more failed fix attempts on the same bug
- Misdiagnosis that led to a wrong fix being deployed
- Violation of a behavioral rule (for example, skipping a required documentation step)
- Security-sensitive mistake (exposed secrets, broken auth)

If yes: add a numbered entry with Date, Severity, What happened, Impact, Root cause, and Rules to Prevent Recurrence. Be honest. Error-log entries exist to prevent recurrence, not to assign blame.

**Feedback loop when an agent makes a mistake:**

1. **Identify** — What went wrong?
2. **Diagnose** — Was it a context, constraint, or correction problem?
   - Context: agent did not know something → update docs
   - Constraint: agent violated a rule → add lint rule or CI check
   - Correction: issue accumulated over time → add periodic audit
3. **Fix** — Have the agent implement the countermeasure in the repository
4. **Verify** — Test that the fix would have prevented the original issue

Fixes go into the repository (docs, linters, tests), not into ephemeral session memory. Always ask: "Will this fix prevent the category of mistake, or just this instance?"

> See also: [`documentation/doc-maintenance.md`](../documentation/doc-maintenance.md)  
> See also: [`practices/feature-lifecycle.md`](feature-lifecycle.md)

---

## State loss and untracked-file handling

**Never delete, move, rename, or overwrite an untracked or gitignored file without explicit user confirmation and a local backup.**

Gitignored means untracked, not disposable. These files exist only locally; if lost, they cannot be recovered from version control.

Before any operation on a gitignored file:
1. Confirm the file is gitignored (for example, `git check-ignore <file>`)
2. If gitignored: stop. Ask the user before proceeding.
3. If proceeding: create a timestamped backup in an archive location first.

"Archive the original" means creating a frozen snapshot — it does NOT mean deleting the working copy. The working copy continues to serve its purpose.

When refactoring documentation: replace content in-place rather than deleting and re-creating, especially for files that may be gitignored.

**Critical project files must never be carelessly deleted.** Whether version-controlled or locally gitignored, before any destructive operation on a file, check whether it is gitignored. If gitignored: stop and confirm with the user. Never delete, move, rename, or overwrite critical files without explicit user confirmation and a local backup.

---

## Verification gaps

**Local tests and production infrastructure verification are two distinct, non-substitutable gates.**

| What local tests prove              | What local tests do NOT prove                  |
|-------------------------------------|------------------------------------------------|
| Code logic is correct               | Cloud resources exist                          |
| Schema validation works             | Permission grants are sufficient               |
| Error handling paths work           | Network connectivity is correct                |
| API contracts are honored           | DNS and TLS are configured                     |
| Auth logic is correct (mocked)      | Real auth tokens are accepted                  |
| Data store queries are well-formed  | Data store tables or indexes exist             |
| Storage operations are correct      | Storage containers exist with correct policies |

The gap between "tests pass" and "production works" is infrastructure. After every deploy introducing new resources, verify those resources exist in production — passing tests are not sufficient.

**Any "user X will see or be notified about Y" feature requires an integration test fired from at least two entry points.** Optional callback handler props are footguns — make them required so the type system catches wiring breaks. Any new persistent store must be added to test fixtures in the same change or the code path is structurally untestable.

**Pagination defaults on detail-page lookups silently return "not found" for entities past page one.** Look up by ID instead of searching a paginated list whenever uniqueness is expected.

**When adding caching or lazy-evaluation to a new code path, verify the cache is active post-deploy.** A test mock cannot tell you whether the external service actually honored the optimization marker.

**Smoke test role-gated features with at least two user roles after deploy.** Trace every navigation entry point across all role combinations, including dual-role users.

**Every page that depends on a record existing must have an actionable empty state.** Query data consumers must differentiate loading, error, and empty states explicitly — never collapse error and loading into a single branch.

> See also: [`practices/testing-principles.md`](testing-principles.md)  
> See also: [`practices/git-workflow.md`](git-workflow.md)

---

## Responsive UI layout

> See also: [`documentation/checklist-protocol.md`](../documentation/checklist-protocol.md) — the full UI review checklist including the mobile/desktop verification gate.

**Every fix must be verified on both viewports before moving on.** A mobile-only fix may have broken desktop. Never add a break or reduce a font size without checking the other viewport.

**Trust the screenshot over DOM measurements.** Font rendering, letter-spacing, decorative elements, and absolute-positioned parents can cause visual overflow that DOM measurements do not capture. If the screenshot shows clipping, there is a real problem even if the DOM reports no overflow.

**Check every location where a pattern was applied, not just the one you fixed.** If you applied a fix pattern once, verify every place that pattern could have been applied.

**Never dismiss a visual artifact without secondary verification.** Sometimes it is a tool artifact; often it is a real bug. Verify with at least one other signal (DOM measurement or accessibility snapshot) before dismissing.

**Default to mobile-only conditional breaks.** Never add an unconditional line break to fix a mobile issue — it applies to all viewport sizes. Use an unconditional break only when the break is intentional at all viewport sizes.

**Common responsive layout anti-patterns and their fixes:**

| Pattern | Symptom | Fix |
|---------|---------|-----|
| Orphaned word at end of heading | Single word alone on last line | Non-breaking space between last two words, or reduce heading size on mobile |
| Mid-word hyphenation at large sizes | Long hyphenated word splits across lines | Reduce mobile font size or use non-breaking hyphen |
| Forced unconditional break on desktop | Awkward break on desktop from a mobile fix | Use a mobile-only conditional break instead |
| Grid forced to 3+ columns on mobile | Cramped, unreadable layout | Use responsive grid: 1 column on mobile |
| Fixed widths that overflow | Element plus padding exceeds viewport | Responsive width: full-width on mobile, fixed on desktop |
| Absolute-positioned overflow | Horizontal scroll, clipped text | Add overflow-x-hidden to root container |
| Text too small for presentation surfaces | Unreadable from a distance | Enforce a minimum readable font size for all body content |
| Multi-column table on mobile | Table overflows or gets cut off | Dual layout: card stack on mobile, table on desktop |
| Large heading font size on mobile | Ugly word breaks on narrow viewport | Step down one or two font sizes on mobile |
| overflow-hidden clipping legitimate content | Text or content clipped within a section | Move overflow-x-hidden to the page root, not individual sections |

**CSS transforms do not move interactive hit areas.** Using a transform to toggle panel visibility on mobile causes the touch/click area to stay at the original position. Use `display:none` or equivalent to fully remove the element from layout and event handling.

**Quick-reference for common responsive layout needs:**

| Need | General approach |
|------|-----------------|
| Keep two words together | Non-breaking space between them |
| Keep hyphenated compound word together | Non-breaking hyphen |
| Line break on mobile only | Conditional break hidden on larger screens |
| Line break on desktop only | Conditional break visible only on larger screens |
| Prevent horizontal overflow on page | overflow-x-hidden on root container |
| Smaller heading on mobile | Responsive font size stepping down on small viewports |
| Stack cards on mobile | Responsive grid: 1 column on mobile, multi-column on desktop |
| Full-width on mobile, fixed on desktop | Responsive width utility |
| Hide decoration on mobile | Visibility utility hidden on small screens |
| Show alternate mobile layout | Dual markup with viewport-conditional visibility |

---

<!-- Assembled from: agent-behavior.md §5 Demand Elegance, agent-behavior.md §8 Core Working Principles, architecture.md §Intended Rate Limits, CLAUDE.md §ERRATA, conventions.md §Python/Backend, conventions.md §TypeScript/Frontend, conventions.md §Tiered staleTime by Data Volatility, conventions.md §Cache Invalidation, conventions.md §Anti-Patterns, core-beliefs.md §Architecture belief 1, core-beliefs.md §Architecture belief 2, core-beliefs.md §Architecture belief 3, core-beliefs.md §Architecture belief 4, core-beliefs.md §Architecture belief 5, core-beliefs.md §Code Quality belief 6, core-beliefs.md §Code Quality belief 7, core-beliefs.md §Code Quality belief 8, core-beliefs.md §Discipline belief 15, core-beliefs.md §Discipline belief 16, debugging.md §4 Anti-Patterns, debugging.md §6 Decision Record, dev-environment.md §Bash Command Safety, dev-environment.md §Known Pitfalls, dev-environment.md §Git Worktrees, ERRATA.md §ERR-001, ERRATA.md §ERR-004, ERRATA.md §ERR-005, ERRATA.md §ERR-007, ERRATA.md §ERR-013, harness-guidelines.md §1 What Is a Harness, harness-guidelines.md §2.4 Mechanical Enforcement Over Convention, harness-guidelines.md §2.6 Iterative Signal-Driven Improvement, harness-guidelines.md §2.9 Critical Files Are Sacred, harness-guidelines.md §5.1 Constraint Hierarchy, harness-guidelines.md §5.2 Deterministic Constraints, harness-guidelines.md §5.3 Structural Constraints, harness-guidelines.md §5.5 Constraint Design Principles, harness-guidelines.md §6.4 Golden Principles, harness-guidelines.md §7 Feedback Loop Protocol, post-deploy-checklist.md §5 ERRATA Check, post-deploy-checklist.md §Debugging-Specific Addendum, security.md §Known Security Gaps, ui-review-checklist.md §2 Common Bad Patterns, ui-review-checklist.md §3 Fix Techniques Cheat Sheet, ui-review-checklist.md §6 Anti-Patterns, data-model.md §Deal Invitations -->
