# AWS deployment patterns

Deploy a FastAPI backend on App Runner and a Next.js frontend on Amplify, wired together through a set of ordered CloudFormation stacks. The most common source of deployment failures is environment-variable placement: inlining secrets in the wrong layer, or setting `NEXT_PUBLIC_*` vars in two places and having one silently shadow the other. Get the env-var split right first, deploy infra in dependency order second, and every other step becomes mechanical.

---

## Environment variables: build-time vs runtime

This is the single most important section in this document. Env-var misplacement causes silent failures that look like CORS errors, auth failures, or blank pages.

| Layer | Canonical source | Do NOT use |
|-------|-----------------|------------|
| **App Runner (backend)** | `backend/apprunner.yaml` → `run.env` (committed to repo) | App Runner Console env vars — they are invisible in `ConfigurationSource: REPOSITORY` mode |
| **Amplify (frontend)** | `frontend/.env.production` (committed to repo) | Amplify Console env vars — they become OS-level overrides that shadow `.env.production` and create a split source of truth |

`NEXT_PUBLIC_*` vars are inlined at build time by Next.js. They must exist in `.env.production` at the moment `npm run build` runs. A var set only in the Amplify Console will either override the repo value (causing invisible divergence between environments) or be missing from the build entirely.

**To change a backend env var:** edit `backend/apprunner.yaml`, commit, push. App Runner auto-redeploys.
**To change a frontend env var:** edit `frontend/.env.production`, commit, push. Amplify auto-redeploys.

### Backend env var inventory

| Variable | Purpose |
|----------|---------|
| `AWS_REGION` | AWS region for all SDK calls |
| `DYNAMODB_TABLE_PREFIX` | Table name prefix (all tables share one prefix) |
| `COGNITO_USER_POOL_ID` | JWT verification pool ID |
| `COGNITO_CLIENT_ID` | Cognito app client |
| `COGNITO_VERIFY_TOKENS` | Set `true` in prod; `false` in tests to bypass JWKS |
| `S3_BUCKET_NAME` | Upload bucket |
| `BEDROCK_MODEL_ID` | AI model ID (e.g., `us.anthropic.claude-sonnet-4-20250514-v1`) |
| `STEP_FUNCTION_ARN` | Async pipeline trigger; empty string = no-op (local dev) |
| `AURORA_ENABLED` | Feature flag, default `false`; app starts without Aurora when disabled |
| `AURORA_HOST` / `AURORA_PORT` / `AURORA_DB` / `AURORA_USER` / `AURORA_PASSWORD` | Aurora credentials when enabled |
| `SES_FROM_EMAIL` | Transactional notification sender |
| `FRONTEND_URL` | Frontend origin for email link construction |
| `CORS_ORIGINS` | JSON array of allowed origins — must include `https://`; e.g., `["https://app.example.com"]` |

### Frontend env var inventory

| Variable | Purpose |
|----------|---------|
| `NEXT_PUBLIC_API_URL` | Backend URL (e.g., `https://api.example.com`) |
| `NEXT_PUBLIC_COGNITO_USER_POOL_ID` | Cognito pool ID; format is `<region>_<poolId>` — not an ARN |
| `NEXT_PUBLIC_COGNITO_CLIENT_ID` | Cognito client ID — separate alphanumeric string; `undefined` here causes `Failed to construct 'URL': Invalid URL` at runtime |
| `NEXT_PUBLIC_COGNITO_IDENTITY_POOL_ID` | Cognito identity pool ID |

---

## AWS account setup

Choose a single region for all resources and lock every Console session to it before creating anything.

1. Enable MFA on the root account: IAM → Dashboard → Add MFA for root user.
2. Create an admin IAM user with `AdministratorAccess` and MFA enabled. Never use root credentials for daily operations.
3. Verify Bedrock model access: Bedrock → Model catalog → open your target Claude model in the Playground. The old "Model access" page has been retired; serverless foundation models are auto-enabled on first invocation. First-time Anthropic users will be prompted for a use-case description during Playground access.

---

## Route 53 DNS setup

Transfer DNS management from your registrar to Route 53 before setting up SSL or custom domains. Domain registration stays at the registrar.

1. Route 53 → Hosted zones → Create hosted zone (Public).
2. Copy the 4 NS records Route 53 generates.
3. Replace existing nameservers at your registrar with all 4 Route 53 nameservers.
4. Verify propagation at [dnschecker.org](https://dnschecker.org/) — typically under 1 hour.

Do not proceed to SSL certificate setup or SES verification until propagation is complete. SSL validation requires working DNS.

---

## CloudFormation stack ordering

Deploy stacks in dependency order. Every output a downstream stack needs must exist before that stack deploys.

| Order | Stack | Purpose |
|-------|-------|---------|
| 0 | `deploy-bucket` | S3 bucket for Lambda deployment packages (versioning enabled) |
| 1 | `cognito` | User pool + app client |
| 2 | `dynamodb` | Application tables + GSIs |
| 3 | `s3` | Upload bucket + policies |
| 3.5 | `rds` | Aurora Serverless v2 — relational/permissions layer |
| 4 | `ses` | Email sender domain identity (deploy before Cognito if SES ARN is imported) |
| 5 | `lambdas` | Async processing functions + IAM roles (requires deploy-bucket) |
| 6 | `step-functions` | Async state machine (requires Lambda ARNs) |
| 7 | `app-runner` | FastAPI backend hosting (requires Cognito + RDS + Step Functions outputs) |
| 8 | `amplify` | Next.js frontend hosting (requires App Runner URL) |

Per-environment parameter files: `infra/parameters/dev.json` and `infra/parameters/prod.json`. Pass outputs between stacks via `Fn::ImportValue` or explicit parameter threading.

### Cross-stack output dependencies

| Source stack | Key outputs | Consumed by |
|-------------|-------------|-------------|
| `deploy-bucket` | `DeployBucketName` | `lambdas` |
| `cognito` | `UserPoolId`, `UserPoolClientId`, `JwksUrl` | `lambdas`, `app-runner`, `amplify` |
| `dynamodb` | Table ARNs | `lambdas` |
| `s3` | `BucketArn`, `BucketName` | `lambdas`, `app-runner` |
| `rds` | `ClusterEndpoint`, `SecretArn` | `app-runner` |
| `lambdas` | Function ARNs | `step-functions` |
| `step-functions` | `StateMachineArn` | `app-runner` |

### Lambda packaging

Before deploying `lambdas`, run the packaging script (`infra/scripts/package-lambdas.sh`). It zips each handler with its dependencies and uploads to the deploy bucket. S3 object versioning means CloudFormation detects new uploads automatically — no manual Lambda console updates needed.

---

## SES email verification

CloudFormation deploys the SES domain identity and configuration set. It does NOT publish DNS records. That requires a manual step after each fresh deploy.

### Deployment sequence

1. Deploy the `ses` CloudFormation stack.
2. In SES → Verified identities → your domain → Authentication tab, copy the 3 DKIM CNAME records.
3. Create those 3 CNAME records in Route 53 (TTL: 300). Wait 5–15 minutes for DKIM status to flip to **Successful**.
4. Add the apex SPF TXT record: `"v=spf1 include:amazonses.com ~all"` (soft-fail at apex — avoids breaking other senders).
5. Add DMARC: TXT at `_dmarc` → `"v=DMARC1; p=none; pct=100; rua=mailto:<report-address>"`. Start at `p=none`. Escalate to `p=quarantine` only after observing clean DKIM + SPF alignment in reports for 1–2 weeks.
6. For MAIL FROM SPF alignment, add MX + TXT records at the `mail` subdomain:
   - MX: `10 feedback-smtp.<region>.amazonses.com`
   - TXT: `"v=spf1 include:amazonses.com -all"` (`-all` is safe on the MAIL FROM subdomain — it only sends via SES)
7. SES starts in sandbox mode. Request production access: SES → Account dashboard → Request production access. Describe transactional use case (invitations, password resets, signup verification). AWS reviews within 24 hours.

### Cognito stack dependency ordering

The `cognito` stack imports the SES domain identity ARN via `Fn::ImportValue`. The hard precondition is DKIM = Successful, not MAIL FROM. Because `BehaviorOnMxFailure: USE_DEFAULT_VALUE` is set, SES falls back gracefully if MAIL FROM is not yet verified. DMARC passes via DKIM alignment regardless.

**Fresh-environment order:** deploy `ses` → publish DKIM CNAMEs → wait for DKIM = Successful → add apex SPF + DMARC → deploy `cognito`. MAIL FROM records are a deliverability optimization to add after the Cognito stack is live.

**Update order (DKIM already verified):** `ses` and `cognito` can deploy back-to-back without a DNS gate.

---

## App Runner setup (FastAPI)

### Python 3.11 critical notes

Python 3.11 on App Runner uses a "revised build" that discards packages installed in the `build` phase. All five points below cause failed deploys if ignored.

1. **Packages installed in `build` are discarded.** Install in `pre-run` only. `apprunner.yaml` must declare `pre-run: pip3 install -r requirements-prod.txt`.
2. **Use `pip3`, not `pip`.** The runtime image only has `pip3` on PATH.
3. **Use `python3 -m uvicorn`, not `uvicorn`.** Pip-installed scripts are not on PATH.
4. **Use a config file (`apprunner.yaml`), not inline Console commands.** The `pre-run` key only exists in the file format, not the inline Console editor.
5. **Use a production requirements file** that excludes `pytest`, `moto`, linters, and other dev-only packages.

### Health check

Set the health check to HTTP protocol with path `/health`, interval 10s, timeout 5s, unhealthy threshold 5. The TCP default will not work for HTTP-based health checks. Expect approximately one health-check request every 10 seconds in logs — this is normal. If only health checks appear in logs with no other traffic, the frontend is not reaching the backend: wrong `NEXT_PUBLIC_API_URL`, CORS misconfiguration, or Amplify deploy still in progress.

### IAM instance role

The instance role must have a trust policy with `tasks.apprunner.amazonaws.com` as the principal (with the `tasks.` prefix). Without this prefix, the role will not appear in the App Runner instance role dropdown.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "tasks.apprunner.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

After creating the role, click the refresh icon in the App Runner Console dropdown before selecting it.

### VPC Connector and NAT Gateway

If App Runner needs to reach Aurora in a private VPC, add a VPC Connector using the same private subnets and security group as Aurora. A VPC Connector routes ALL outbound traffic through the VPC. Private subnets have no internet access by default — deploy a NAT Gateway or every outbound call (Cognito JWKS, SES, Bedrock) will fail, causing 401 errors on all authenticated requests.

### Baseline sizing

| Setting | Value |
|---------|-------|
| CPU | 1 vCPU |
| Memory | 2 GB |
| Max concurrency | 100 |
| Min instances | 1 |
| Max instances | 4 |
| Health check path | `/health` |
| Health check interval | 10 s |

Auto-deploy from GitHub source: 3–5 minutes per deploy.

---

## Amplify setup (Next.js)

### SSR detection pitfalls

1. **Use `next.config.mjs` (or `.js`), not `next.config.ts`.** Amplify's framework detector only recognizes `.js`/`.mjs` config files. If it finds only `.ts`, it classifies the app as static "Web" and the deployed site shows a placeholder after a successful build.
2. **Check "My app is a monorepo"** and set the monorepo root directory to `frontend` during the "Add repository" step. This is where SSR detection happens.
3. **If Amplify misclassified the framework,** fix via CLI and redeploy:
   ```bash
   aws amplify update-branch --app-id <APP_ID> --branch-name main \
     --framework "Next.js - SSR" --region <region>
   ```
4. **Node.js 20+ required.** Add `.nvmrc` with `20` in `frontend/`. Amplify dropped Node.js 18 support in Sep 2025.
5. **`baseDirectory: .next`** is required for all Next.js 14+ builds in `amplify.yml`.
6. **Service role required for WEB_COMPUTE.** Amplify SSR needs an IAM service role with `amplify.amazonaws.com` trust. Create it during setup or Amplify will prompt; without it, SSR compute cannot run.

### amplify.yml (monorepo format)

```yaml
applications:
  - appRoot: frontend
    frontend:
      phases:
        preBuild:
          commands: ["npm ci"]
        build:
          commands: ["npm run build"]
      artifacts:
        baseDirectory: .next
        files: ["**/*"]
      cache:
        paths: ["node_modules/**/*", ".next/cache/**/*"]
```

### Env vars

Do NOT set `NEXT_PUBLIC_*` vars in the Amplify Console. Set them in `frontend/.env.production` (committed to repo). Console vars become OS-level overrides that shadow `.env.production`, creating a split source of truth and making it impossible to reproduce builds locally.

Auto-deploy from GitHub: 2–4 minutes per deploy. Amplify only triggers on changes inside `frontend/` — backend-only commits to `main` do not trigger a frontend redeploy.

---

## Custom domains

### App Runner (API subdomain)

1. App Runner → your service → Custom domains → Link domain: `api.<your-domain>`.
2. Create a **CNAME** record in Route 53: name `api`, value `<default>.awsapprunner.com`. Do NOT use an A record — A records expect IP addresses.
3. Wait for status Active (5–30 minutes).
4. Update `NEXT_PUBLIC_API_URL` in `frontend/.env.production` from the default App Runner URL to `https://api.<your-domain>`, commit, and push. Amplify auto-redeploys.

### Amplify (root domain)

Amplify → Hosting → Custom domains → Add domain → select your hosted zone → map root domain to branch `main`. Amplify auto-creates the required Route 53 records. Wait for status Available (10–30 minutes).

---

## Infrastructure change gate

Every AWS surface change — new DynamoDB table or GSI, Aurora column/migration, new IAM policy, CloudFormation template edit, env var addition, Cognito or SES config change, new Lambda/Step Function/SQS/SNS, App Runner or Amplify config edit — requires plan-mode approval before implementation. See [`infra-aws.md`](infra-aws.md) for DynamoDB and Aurora patterns; see `core/practices/infra-management.md` for the approval protocol.

**Net infra footprint check:** at every session end, enumerate every AWS surface touched and record the delta in the commit message and exec plan, even when the delta is zero. Format:

> **Net infra footprint:**
> | Surface | Change |
> |---------|--------|
> | DynamoDB | + `<table_name>` table (PK/SK pattern), 1 GSI |
> | IAM | App Runner instance role gains `<action>` on `<resource>` |
> | env | + `<VAR_NAME>` (App Runner) |

The two historical failure classes this prevents: (1) DynamoDB tables declared in code but never created in CloudFormation — symptoms present as CORS or 500 errors that mask the missing-table root cause; (2) Cognito email config missing SES `SourceArn` wiring — verification emails land in spam.

---

## Post-deploy smoke test

Run this after every deploy. Both Amplify and App Runner must finish deploying before starting.

1. `GET https://<api-domain>/health` — expect `{"status": "healthy"}`.
2. Navigate each primary page in the app and verify no 5xx errors.
3. Test with all distinct user roles — they hit different authorization paths and endpoints.
4. Run Playwright smoke suite:
   ```bash
   PLAYWRIGHT_BASE_URL=https://<app-domain> npx playwright test --project=smoke
   aws s3 cp test-results/ s3://<test-reports-bucket>/$(date +%Y-%m-%d)/smoke/ \
     --recursive --region <region>
   ```
5. Verify DynamoDB table existence matches code references:
   ```bash
   grep -roh 'settings\.table_name("[^"]*")' backend/app/services/ | sort -u
   aws dynamodb list-tables --region <region> --output text
   ```
   Any table in code but not in AWS is a launch blocker. The symptom is often a 500 or CORS error that makes the missing table invisible.
6. Store test reports to S3 (PAY_PER_REQUEST bucket, no public access, 90-day lifecycle rule).

---

## Deployment timing reference

| Service | Deploy trigger | Typical time | How to check |
|---------|---------------|-------------|--------------|
| App Runner (backend) | Push to `main` (auto-deploy from GitHub) | 3–5 min | App Runner console → Service → Deployments |
| Amplify (frontend) | Push to `main`, changes inside `frontend/` | 2–4 min | Amplify console → App → Build history |
| CloudFormation | Manual `aws cloudformation update-stack` | 1–10 min | CF console → Stack → Events |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| App Runner `CREATE_FAILED` "connection arn is invalid" | CodeStar connection uses `codeconnections` ARN format; App Runner CF expects `apprunner` ARN format | Use Console setup instead of CloudFormation for App Runner |
| Instance role not in App Runner dropdown | Trust policy missing `tasks.` prefix | Edit trust policy principal to `tasks.apprunner.amazonaws.com`, then refresh dropdown |
| `pip: command not found` during build | Python 3.11 only has `pip3` on PATH | Use `pip3` in all `apprunner.yaml` commands |
| `uvicorn: executable file not found` on start | Pip-installed scripts not on PATH | Use `python3 -m uvicorn` |
| `No module named uvicorn` (or any dep) | Python 3.11 revised build discards `build`-phase packages | Move install to `pre-run` in `apprunner.yaml`; set config source to "Use a configuration file" |
| App Runner env vars missing after redeploy | REPOSITORY config ignores Console env vars | All vars must be in `apprunner.yaml` under `run.env` |
| Only health-check requests in App Runner logs | Frontend not reaching backend | Check `NEXT_PUBLIC_API_URL`, CORS config, and whether Amplify deploy is still running |
| Amplify shows placeholder after green build | Framework detected as "Web" (`.ts` config or missing monorepo checkbox) | Fix via `aws amplify update-branch --framework "Next.js - SSR"`, redeploy, ensure `next.config.mjs` exists |
| Amplify `NEXT_PUBLIC_*` vars not working | Vars set in Console override `.env.production` with wrong values | Remove all `NEXT_PUBLIC_*` from Amplify Console; use `frontend/.env.production` only |
| CORS errors on all requests | `CORS_ORIGINS` missing `https://` or wrong format | Must be a JSON array: `["https://your-domain.com"]` |
| 401 on all authenticated requests | No NAT Gateway when VPC Connector is attached | VPC Connector routes all outbound through VPC; private subnets need a NAT Gateway to reach Cognito JWKS |
| `DynamoDB ValidationException: table does not have the specified index` | GSI added to CloudFormation but stack never updated | Run `aws cloudformation update-stack` for the `dynamodb` stack |
| CORS errors that look like a CORS config problem | Missing DynamoDB table | Run `aws dynamodb list-tables` and diff against code references before touching CORS config |
| Custom domain `ERR_NAME_NOT_RESOLVED` | Wrong DNS record type | App Runner custom domains need a CNAME, not an A record |
| `NEXT_PUBLIC_COGNITO_USER_POOL_ID` causes `Failed to construct 'URL': Invalid URL` | Wrong ID format or `undefined` | Pool ID format is `<region>_<poolId>` — not an ARN; Client ID is a separate alphanumeric string |
| SES not sending | Account in sandbox mode | Request production access in SES Console → Account dashboard |

---

## Cross-references

- See [`practices/infra-aws.md`](infra-aws.md) for DynamoDB table patterns, Aurora Serverless v2 setup, S3 presigned URL patterns, and Bedrock invocation.
- See [`practices/security-aws.md`](security-aws.md) for Cognito JWT Depends() patterns, Google SSO wiring, role-based access, and secrets inventory.
- See [`practices/debugging-fullstack.md`](debugging-fullstack.md) for App Runner health-check diagnostics, CloudWatch log recipes, and frontend-backend wiring failures.
- See [`practices/testing-fullstack.md`](testing-fullstack.md) for `COGNITO_VERIFY_TOKENS=false`, moto DynamoDB mocks, and Playwright E2E setup.
- See `core/practices/infra-management.md` for the plan-mode approval protocol and net infra footprint check rule.
- See `core/practices/git-workflow.md` for the `dev` → `main` commit discipline.

---

<!-- Assembled from: agent-behavior.md §10.5, §11; debugging.md §3 Deployment Debugging Checklist, §5.1 AWS Amplify Hosting; deployment.md §Critical: Where Environment Variables Live, §AWS Account Setup, §Route 53 DNS Setup, §Deploy CloudFormation Stacks, §SES Email Verification, §App Runner Setup (Console Path), §Amplify Setup (UI Path), §Custom Domain Configuration; infrastructure.md §CloudFormation Stacks (11 stacks), §CloudFormation Stack Output Dependencies, §Hosting, §App Runner Configuration, §Lambda Functions, §Backend, §Frontend (.env.local); post-deploy-checklist.md §0 Net Infra Footprint Check, §1 Infrastructure Verification, §2 Production Smoke Test, §Appendix: Deployment Timing -->
