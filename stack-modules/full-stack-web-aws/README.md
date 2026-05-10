# Stack: full-stack-web-aws

**Status:** v0.1.0 (bootstrapped 2026-05-10 from a single source project).
**Version:** v0.1.0. See [v0.1.0/STACK.md](v0.1.0/STACK.md) for the doc index and [v0.1.0/STACK-CHANGELOG.md](v0.1.0/STACK-CHANGELOG.md) for the lineage.

## Archetype definition

Projects matching this archetype use:

- **Backend:** a Python or TypeScript service framework (FastAPI, Express, Next.js API routes, Django, Flask) deployed as a containerized HTTP service.
- **Frontend:** a React-based web UI (Next.js, Vite + React, Remix), TypeScript-first, deployed as a static or hybrid build.
- **Cloud:** AWS as the primary deployment target. Common services: App Runner, Lambda, ECS, DynamoDB, Aurora, RDS, S3, Cognito, SES, Bedrock, MediaConvert, Step Functions, CloudFront, Route 53, CloudFormation, Amplify Hosting.
- **Local environment:** Windows or POSIX, with Python virtual environments and Node version managers (conda, nvm, asdf).

A project is in this archetype if it ships ≥2 of the above categories. Pure-backend services (no frontend) belong in a separate `service-aws` archetype; pure-frontend SPAs (no backend) belong in `web-spa`.

## Layout (when populated)

```
full-stack-web-aws/
├── README.md
├── STACK.md                       # Index — what's here, current version
├── STACK-CHANGELOG.md             # Evolution log
├── practices/                     # Stack-level practices
│   ├── debugging-fullstack.md     # CORS, service worker, frontend-backend wiring
│   ├── testing-fullstack.md       # pytest/vitest/Playwright patterns, mocking discipline
│   ├── deployment-aws.md          # App Runner, Lambda, Amplify deploy patterns
│   ├── infra-aws.md               # CloudFormation patterns, IAM ordering, env-var wiring
│   └── dev-environment-fullstack.md  # Python venv, conda+npm interop, Windows pitfalls
├── documentation/
│   └── doc-architecture-fullstack.md  # What docs a full-stack project needs
└── templates/
    └── project-scaffold/          # Stack-augmented version of core/templates/project-scaffold/
```

## Bootstrap

Until populated by the bootstrap extraction (`Downloads/stack_extraction_prompt.md`), this directory holds only this README and a `.gitkeep`. The first extraction reads from a source project's refactored project-docs and produces v0.1.0.
