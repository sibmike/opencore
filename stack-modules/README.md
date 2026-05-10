# Stack Modules

A stack module is a CORE-adjacent layer that captures wisdom common to projects of the same technical archetype. Stack content sits between universal CORE practices and project-specific PROJECT docs.

## Why this layer exists

CORE forbids any specific technology, framework, vendor, or service name. That keeps it portable but leaves a real gap: every new full-stack-web-on-cloud project re-discovers the same patterns (CORS wiring, service worker debugging, framework testing discipline, language-family pitfalls) that older projects in the same archetype already paid for.

Stack modules absorb that wisdom. A new project forking CORE also picks up the matching stack module and arrives with the framework-level patterns already in place.

## The 4-layer model

```
CORE                (universal: HOW to work)
  ↓
STACK               (per archetype: full-stack-web-aws, cli-go, data-pipeline, ...)
  ↓
PROJECT             (this codebase: WHAT was built)
  ↓
USER                (the human)
```

A project's docs typically include forks of all three upper layers:

```
<project>/docs/
├── core-practices/      # forked from core/, drifts as project demands
├── stack-practices/     # forked from core/stack-modules/<archetype>/, drifts
└── ...                  # project-specific docs
```

## What belongs in a stack module

Stack content names the archetype's technical surface concretely:

- Programming language families (Python, TypeScript, Go).
- Framework names (FastAPI, Next.js, React, Django, Flask, Express, Gin).
- Cloud vendor + service categories (AWS App Runner, Lambda, DynamoDB, Aurora; GCP Cloud Run, Firestore; Azure App Service).
- Generic technical patterns scoped to the archetype (CORS wiring, service worker invalidation, conda+npm interop on Windows, framework-DI patterns).

## What does NOT belong in a stack module

- Specific table names, endpoint paths, schema fields, or business-domain terms (those are PROJECT).
- Universal practices that apply to any archetype (those are CORE).
- Personal machine paths, shell preferences, individual workflow habits (those are USER).
- Specific milestone IDs, ERRATA IDs, or named incidents from a source project.

## Archetypes

Each archetype lives in its own subdirectory and versions independently. Current archetypes:

- [full-stack-web-aws/](full-stack-web-aws/) — Python or TypeScript backend + React/Next.js frontend + AWS services.

New archetypes get added when at least two projects in a previously-uncovered archetype produce convergent stack-level drift across their dreams.

## Versioning

Each stack module follows independent semver. A project records both its CORE version and its STACK version in `<project>/docs/LINEAGE.md`:

```
core: v0.1.2
stack/full-stack-web-aws: v0.1.0
```

## Evolution

Stack modules evolve through the same dream-and-delta machinery as CORE, with one difference: the divergence review compares the project's `docs/stack-practices/` against `core/stack-modules/<archetype>/` and the gate is lower (≥2 engineers from ≥2 projects of the same archetype, vs CORE's ≥3 from ≥2 projects of any archetype). See [`../evolution/stack-divergence.md`](../evolution/stack-divergence.md).

## Bootstrapping a stack module from existing project docs

When a CORE has been forked but no stack module exists yet for the archetype, the existing project's PROJECT docs already carry stack-level wisdom mixed in. Extracting it is a one-time bootstrap, separate from ongoing evolution. See the bootstrap orchestrator prompt at `Downloads/stack_extraction_prompt.md`.
