# Writing Style

When to update: when a new structural convention is adopted project-wide, when a new doc type is introduced, or when a rule in this doc conflicts with observed agent behavior.

---

## Document Design Rules

Apply these rules to every doc you write or maintain.

1. **Single responsibility.** Each doc covers one domain. A doc that needs to reference another to be usable within its own scope should be split.
2. **Living documents.** Update in place on every relevant commit. Never append changelogs — docs are snapshots, not logs.
3. **Concise by default.** If a section exceeds roughly 30 lines, the detail belongs in code comments or a sub-doc.
4. **Machine-parseable structure.** Use consistent heading levels, tables, and code blocks. Avoid prose paragraphs where a table or list works better.
5. **Cross-reference with relative links.** When one doc references another, link — do not duplicate content.
6. **Freshness markers.** Every doc starts with a "When to update" note specifying the triggers for an update.
7. **Index files for subdirectories.** Every subdirectory contains an index file cataloging its contents with verification status.

> See also: [documentation/doc-maintenance.md](../documentation/doc-maintenance.md)

---

## Voice and Density

Every word must carry weight.

- **Imperative second-person.** Write "Check the logs first." Not "It is recommended that one should check the logs."
- **Principle first, example second.** State the rule. Follow with one anonymized example only if needed. Never bury the rule under setup.
- **No hedging.** Cut "perhaps", "might", "could potentially", "in general", "tend to", "often", "usually" unless uncertainty is itself the point.
- **No filler openings.** Cut "It is important to note that", "In order to", "There are several ways to". Start with the verb or the subject.
- **One idea per sentence.** Split compound sentences. Short sentences are not less rigorous — they are easier to act on.
- **Bullets for lists, prose for sequence.** A 4-step protocol is bullets. A causal chain is prose.
- **Anonymized examples.** Never name a specific project, table, service, runtime, or person in reusable docs. Use "the data store", "the runtime", "the service", "the consumer".
- **No needless meta.** Do not preface sections with "This section covers...". The heading does that.
- **Test every paragraph by deletion.** If the doc loses nothing load-bearing when you cut a paragraph, cut it.

---

## README Structure

Every project README should contain these sections in order.

### License

Include a License section, even if it is one sentence. Contributors need immediate clarity on usage terms.

### Prerequisites

List every prerequisite explicitly before any setup steps:

- Required runtimes and their minimum versions
- Required package managers or environment managers
- Required external service accounts and the specific permissions needed

Missing this section causes wasted setup time from version mismatches or missing credentials.

### Quick Start

Show commands for each tier in clearly labeled blocks. For a full-stack project, each tier runs in its own terminal:

```
# Backend
cd backend && <install command> && <start command>

# Frontend (separate terminal)
cd frontend && <install command> && <start command>
```

Keep Quick Start to the minimum needed to reach a running local dev server. Defer advanced options (containerized setups, environment variable details) to subsequent sections.

### Testing

For each tier, document:

1. The exact command to run the full test suite.
2. The command to generate a coverage report.
3. Current test counts and coverage percentage so contributors have a baseline to maintain.

---

## Naming Conventions

Define explicit naming conventions for every artifact type in the codebase — source files, test files, API fields, data store keys, style variables — and document them in a lookup table.

Rules:
- Test files match their source file name plus a `.test` suffix.
- Use consistent prefixes or suffixes for typed artifacts (hooks, services, utilities) so the role of any file is visible from its name alone.
- Apply language-appropriate casing consistently: snake_case for one layer, camelCase for another, kebab-case for a third — never mix within a layer.

> See also: [practices/anti-patterns.md](../practices/anti-patterns.md)

---

## Agent Legibility

Optimize code and docs for agent comprehension, not just human readability.

- **Consistent naming patterns reduce ambiguity.** Agents pattern-match harder than humans — inconsistent names multiply lookup failures.
- **Predictable file locations.** An agent should not have to search for where something lives. Put things where the name of the directory predicts they will be.
- **Explicit over clever.** Avoid metaprogramming, dynamic imports, or patterns that require deep context to understand. A reader should be able to trace execution without running the code.
- **Favor stable, composable dependencies.** Technologies with broad documentation representation are easier for agents to model than exotic or proprietary ones. When an external library's behavior is opaque, consider reimplementing the needed subset with full test coverage rather than fighting upstream surprises.

> See also: [practices/architecture-principles.md](../practices/architecture-principles.md)

---

## Encoding Judgment

Some engineering wisdom cannot be expressed as lint rules: "prefer boring technology," "validate at boundaries," "when in doubt, add a test."

Capture this wisdom as a small set of core beliefs — opinionated, numbered directives that guide decision-making when no specific rule applies. Keep them short. Write them as directives, not explanations. They change rarely — these are team values, not implementation details.

> See also: [practices/session-protocol.md](../practices/session-protocol.md)

---

<!-- Assembled from: README.md §License, README.md §Development > Prerequisites, README.md §Development > Quick Start, README.md §Testing, conventions.md §Naming Conventions, harness-guidelines.md §2.5 Design for Agent Legibility, harness-guidelines.md §2.10 Encode Judgment Not Just Rules, harness-guidelines.md §4.3 Document Design Rules -->
