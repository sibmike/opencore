# Architecture Principles

> See also: [`practices/anti-patterns.md`](anti-patterns.md) — the failure modes these principles prevent.
> See also: [`practices/infra-management.md`](infra-management.md) — infrastructure decisions that flow from these principles.
> See also: [`practices/security-principles.md`](security-principles.md) — authorization patterns referenced in §Authorization.

---

## Storage Selection

Choose storage technology based on access-pattern fit for each entity class, not uniformity.

Match the store to the entity:

- Use a key-value / document store for content entities fetched by a known key, write-heavy event streams, and companion documents (analysis or generated output attached to a primary record). Prefer async drivers.
- Use a relational store for permission and membership entities that require joins, counts, and complex filters across multiple entity types.
- Enforce a single-service boundary: only one service module talks to each data store. This keeps schema migrations, driver upgrades, and query optimization contained.
- Choose multi-table design over single-table design when entities have divergent access patterns and different scaling profiles. Accept the operational cost of more tables in exchange for clarity.
- Name tables with a consistent environment prefix so the same schema coexists across dev, staging, and prod without collision.

Each store should do one thing it does best. Do not force a relational model onto a document store, or vice versa.

> See also: [`practices/infra-management.md`](infra-management.md) — provisioning and migration discipline for data stores.

---

## Backend Layering

Enforce a strict, unidirectional dependency hierarchy:

```
transport layer (HTTP handlers)
    → business logic layer (services)
        → data access layer
```

Services must never import from the transport layer. Cross-cutting concerns — authentication, configuration, database connections — enter only through the framework's dependency injection mechanism, not through direct imports. This keeps them replaceable and mockable in tests.

> See also: [`practices/testing-principles.md`](testing-principles.md) — why injectable dependencies are a prerequisite for unit tests.

---

## Error Handling

Domain and service layers raise typed domain exceptions. A single global exception handler at the transport boundary maps domain exceptions to protocol-level error codes (HTTP status, gRPC code, etc.).

This keeps business logic decoupled from transport concerns. The mapping is auditable in one place.

---

## Async Pipeline Design

**Two-phase upload.** Issue a pre-authorized slot for the client to write directly to object storage, then require an explicit confirm call to transition the record from `uploading` to `processing`. This prevents orphaned records from abandoned uploads.

**Polling discipline.** Use exponential backoff with a hard timeout ceiling rather than fixed-interval polling. Define explicit stop conditions — success, error, timeout, component unmount — so the polling loop has a finite lifecycle even when the backend is stuck.

**Companion-document pattern.** Store generated analysis as a sibling record in the same partition as the source document: same primary key, distinct sort-key suffix. A single query with a sort-key prefix filter retrieves both atomically. Replacing the analysis requires only overwriting the sibling at the same key.

---

## Document-Store Schema Patterns

### Prefix discipline for sibling items

When a partition holds both primary entity rows and metadata or counter siblings, use a sort-key prefix for siblings that is lexicographically distinct from primary entity keys (e.g., `META#` vs `ENTITY#`). This ensures prefix-filtered list queries cannot accidentally return sibling items.

Store notification cooldown state in a dedicated sibling item rather than on the primary entity row, so cooldown reads and writes are isolated from entity mutations.

Guard against missing counter items by returning a zeroed counter when the item does not exist. Treat missing as zeroed.

### Conditional upsert — phantom-row guard

When using upsert behavior (item created if not exists), add a condition expression that requires the primary key to exist before the update. This prevents creating a phantom row for a key that does not exist in the table. This is especially important when the caller cannot guarantee the item exists before the update.

### Source-agnostic schema design

When a table will be populated by multiple producers — manual input today, an automated process later — define the item shape in terms of the consumer's access patterns, not any producer's internal format. Producers write the same shape regardless of their origin. This decouples rendering code from data-source identity and allows producer migrations without frontend changes.

---

## Analytics in a Document Store

For write-heavy, per-entity analytics:

- Store raw events with a time-sortable key (e.g., `EVENT#{ISO_timestamp}#{uuid}`).
- Maintain a sibling aggregate item updated atomically via incremental add expressions for numeric counters and map-attribute expressions for keyed values. This avoids a full scan on every dashboard read.
- The read endpoint reads only the aggregate item for counts — O(1).
- For time-series data, query raw events with a bounded date range (e.g., last 30 days maximum) to prevent unbounded scans as a partition grows.
- Use a distinct key prefix for the aggregate item so prefix-filtered queries on raw events cannot accidentally return it.

---

## Authorization Patterns

**Short-circuit for globally-accessible resources.** When a resource has a publicly accessible mode (e.g., a published flag that grants access to anyone), check that flag at the earliest possible layer before executing an expensive multi-table permission join. The flag check is O(1); the full join is O(n). This avoids unnecessary load for the common public-access case and keeps the permission service simpler.

**Denormalized feed projection.** For a browsable feed of gated content, maintain a denormalized projection table or materialized view that stores only the columns needed for listing queries. Apply partial indexes on the most selective filter conditions to keep feed queries fast as the dataset grows. Include routing keys in the projection so click-throughs do not require a separate lookup.

> See also: [`practices/security-principles.md`](security-principles.md) — the broader access-control model that these query patterns serve.

---

## Type Definitions and API Boundaries

Keep canonical type definitions in source code — typed interfaces for the frontend, schema classes for the backend — not in documentation. Documentation that duplicates type definitions drifts and misleads.

Enforce a naming convention boundary at the API layer: use snake_case in backend models and camelCase in frontend models. A single conversion layer at the API client handles the translation. Neither side needs to be aware of the other's convention.

When documenting a data model, reference source file paths rather than copy-pasting schema definitions.

---

## Responsive Breakpoints

Define named responsive breakpoints in the architecture document with explicit component behavior at each size, not just pixel widths. Record the detection hook name alongside the breakpoint definitions so all contributors know the single source of truth for viewport classification and do not duplicate media-query logic across components.

---

<!-- Assembled from: architecture.md §Core Architecture Principle, architecture.md §Backend Layering, architecture.md §Error Handling, architecture.md §AI Analysis Pipeline, architecture.md §Frontend Responsive Breakpoints, data-model.md §Investor Pool + Matches (M14, PS-27 / DD-34), data-model.md §Design Philosophy, data-model.md §Founder Updates (M13, PS-26), data-model.md §Analytics, data-model.md §Key Permission Queries, data-model.md §Canonical Type Definitions -->
