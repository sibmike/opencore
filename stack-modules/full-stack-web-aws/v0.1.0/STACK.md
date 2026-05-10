# STACK: full-stack-web-aws

Versioning: semver. Current version: 0.1.0.

## Practices
- [dev-environment-fullstack](practices/dev-environment-fullstack.md) — Python venv + Node + Docker Compose local AWS stand-ins + git-worktree constraints (frontend cannot run from worktree) + Windows pitfalls.
- [debugging-fullstack](practices/debugging-fullstack.md) — CORS-vs-missing-infra diagnosis, TanStack Query three-branch guard, App Runner operational debugging, half-built feature wiring audit.
- [testing-fullstack](practices/testing-fullstack.md) — pytest+pytest-asyncio+moto async fixtures, aioboto3 boto3 mock casing, Cognito token bypass, vitest+RTL+MSW v2, Playwright E2E, screenshot pipeline.
- [deployment-aws](practices/deployment-aws.md) — App Runner + Amplify deploy, NEXT_PUBLIC build-vs-runtime env vars, CloudFormation ordered multi-stack deploy, SES DKIM, custom domains, post-deploy smoke test.
- [infra-aws](practices/infra-aws.md) — DynamoDB multi-table patterns (versioned config, analytics aggregates, conditional upsert), Aurora Serverless v2, S3 presigned URLs, Lambda packaging, SES, Bedrock invocation + caching, CloudFormation stack architecture, IAM ordering.
- [security-aws](practices/security-aws.md) — Cognito JWT auth, role-based FastAPI dependency factory, Google SSO Cognito wiring, pre-signed URL discipline, secrets inventory, known security gaps, Bedrock IAM scoping.
- [ui-review-frontend](practices/ui-review-frontend.md) — Next.js + Tailwind responsive audit protocol: desktop 1280×900 + mobile 375×812, 10 bad patterns with fixes, fix techniques cheat sheet, pre-commit verification script.
- [conventions-fullstack](practices/conventions-fullstack.md) — Backend Python/Pydantic/aioboto3 + frontend TypeScript/Next.js/Tailwind + TanStack Query patterns (key factories, cache invalidation, exponential-backoff polling) + naming + anti-patterns.

## Documentation
- [doc-architecture-fullstack](documentation/doc-architecture-fullstack.md) — Doc set a FastAPI + Next.js + AWS project needs; cross-doc wiring; README three-file convention; exec-plan lifecycle; maintenance cadence; plan-file discipline; quality tracking format.

## Compatibility
- Forks alongside CORE v0.1.2 or later.
- Archetype: full-stack-web-aws (Python or TypeScript backend + React/Next.js frontend + AWS services).

## Versioning policy
- Patch: typo fixes, formatting, clarifications.
- Minor: new sections within existing docs, new examples, deepened guidance.
- Major: new docs, removed docs, contradictory updates.

## Lineage
v0.1.0 bootstrapped from a source project on 2026-05-10. See STACK-CHANGELOG.md.
