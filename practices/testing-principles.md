# Testing Principles

## The Contract

Tests are not a quality gate bolted on after coding — they are the specification. Write tests that describe behavior: what the system produces, what the user sees, what an API returns. Never test internal state or implementation details.

Name every test so a failing assertion tells you exactly what broke without reading the body. If the failure message requires source-diving, the name is wrong.

Every public API endpoint has a test. Every new feature ships with tests. No exceptions.

> See also: [feature-lifecycle.md](feature-lifecycle.md) — tests are a required exit criterion for every feature.

## Coverage Policy

Enforce a minimum line-coverage floor (80% is a reasonable starting point) via CI so regressions are caught automatically. That floor is a floor, not a target.

Identify the paths where a defect is catastrophic — authentication flows, data-mutation endpoints, access-control boundaries, public-facing entry points — and require 100% coverage on those paths without exception. The floor lets speed win elsewhere; the 100% mandate ensures the highest-consequence paths are never skipped.

> See also: [security-principles.md](security-principles.md) — access-control boundaries are the overlap between security and testing.

## Test Level Selection

Match the test type to the scope of the claim:

- **Unit** — business logic in isolation. One function, one class, deterministic inputs, no I/O.
- **Integration** — service and API boundaries. Verify the contract between two real components.
- **End-to-end** — critical user journeys. Prove the whole system assembles correctly.

Do not push integration concerns into unit tests by mocking everything. A test that mocks the persistence layer, the network layer, and the service layer is not testing anything real. When a test requires so many mocks that the test code is larger than the production code, it is testing at the wrong level.

> See also: [anti-patterns.md](anti-patterns.md) — over-mocking is called out explicitly.

## One Assertion Concept Per Test

Each test fails for exactly one reason. "One assertion" is too strict as a rule — a single concept may require multiple `assert` calls. The constraint is: one failing reason. If a test can fail for two independent reasons, split it.

## Mocking Discipline

Mock at the boundary of the system under test, not inside it. Mock external services, not internal modules.

When using an API mock layer in tests, configure the mock server to error on unhandled requests rather than silently ignoring them. Silent pass-through hides missing handler coverage and produces false-green tests. Forcing an error on every unmatched request ensures every test explicitly declares its dependencies.

## Fixture and Infrastructure Structure

Structure test infrastructure around five concerns:

1. **One mock handler file per API domain.** Ownership stays clear; changes to one domain don't cascade across unrelated test files.
2. **A per-test error-override helper.** Tests that exercise error paths shouldn't need to replace the entire handler — a targeted override is cleaner and more explicit.
3. **Factory functions for every major data shape.** Test data stays consistent across tests; mutations from defaults are declared explicitly, not embedded in ad-hoc object literals.
4. **A shared render or invocation wrapper.** Resets stateful dependencies (client caches, retry config, auth state) between tests so test order doesn't matter.
5. **A central setup file.** Boots and tears down the mock server, registers custom assertion matchers, and runs once for the suite — not once per file.

Organize mock response payloads (especially for external services like AI APIs) as checked-in fixture files rather than inline strings. Inline strings drift; files are diffable and reviewable.

## Summary of Non-Negotiables

- Test behavior, not implementation.
- One concept per test.
- Test at the right level — unit for logic, integration for endpoints, end-to-end for journeys.
- Name every test so a failure is self-explanatory.
- Every new feature ships with tests.
- Mock servers error on unhandled requests.
- 100% coverage on critical paths; enforce a floor on everything else.

> See also: [debugging.md](debugging.md) — a failing test is the entry point for the debugging protocol.
> See also: [documentation/checklist-protocol.md](../documentation/checklist-protocol.md) — test passage is a checklist item before merge.

<!-- Assembled from: core-beliefs.md §Testing (beliefs 9–11), testing.md §Coverage Targets, testing.md §Test Fixtures, testing.md §Stack (onUnhandledRequest discipline), testing.md §Test Infrastructure, testing.md §Testing Principles -->
