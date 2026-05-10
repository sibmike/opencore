# AWS Infrastructure Patterns

> Cross-references: `core/practices/infra-management.md` — plan-mode approval gate and net infra footprint check.
> Cross-references: `core/practices/architecture-principles.md` — storage selection and document-store schema patterns.
> Sister stack docs: `practices/deployment-aws.md`, `practices/security-aws.md`, `practices/conventions-fullstack.md`.

---

## DynamoDB: Multi-Table Design

Choose one table per entity class with a divergent access pattern. Do not force every entity into a single table — the operational clarity of named tables outweighs the query convenience of table sharing when entities scale independently.

**Table naming.** Prefix every table with `{TABLE_PREFIX}_{entity}`, where `TABLE_PREFIX` is an env var (e.g., `myapp`). This pattern lets dev, staging, and prod coexist in the same account without collision and makes IAM resource patterns unambiguous: `arn:aws:dynamodb:*:*:table/myapp_*`.

**Key schema.** Use composite PK+SK with a typed prefix on every value:

```
PK = ENTITY#{id}
SK = RECORD#{sub_id}   or   META#<purpose>   or   PROFILE
```

Prefix discipline prevents list queries (SK `begins_with RECORD#`) from accidentally returning sibling items (SK `META#...`). Define all prefix constants in a single module.

**Async driver.** Use `aioboto3` for all DynamoDB operations inside async FastAPI handlers. Never use sync `boto3` in an async context — it blocks the event loop.

**BillingMode.** Default to `PAY_PER_REQUEST` for new tables. Switch to provisioned only after profiling steady-state traffic reveals a cost advantage.

**PITR.** Enable Point-In-Time Recovery on prod tables. Use a `!If [IsProd, true, false]` condition in CloudFormation to gate it:

```yaml
PointInTimeRecoverySpecification:
  PointInTimeRecoveryEnabled: !If [IsProd, true, false]
```

**GSI deployment trap.** Adding a GSI to a CloudFormation template does not apply it to the live table. Run `aws cloudformation update-stack` for the DynamoDB stack, or add the index manually via the console. A query against a non-existent GSI throws `ValidationException: The table does not have the specified index` — this is a deployment gap, not a code bug.

---

## DynamoDB: Versioned Config Items

Store mutable configuration that must be auditable — rubric definitions, feature flags, scoring criteria — as versioned items under a single partition:

```
PK = CONFIG#{category}
SK = V#{ISO_8601_timestamp}
```

Retrieve the current version with a reverse scan and `Limit=1`:

```python
resp = await table.query(
    KeyConditionExpression=Key("PK").eq(f"CONFIG#{category}"),
    ScanIndexForward=False,
    Limit=1,
)
```

Retrieve a pinned version with `GetItem` using the exact timestamp. Retrieve the full audit trail with an unlimited reverse scan. Never overwrite existing items — append a new `SK=V#{timestamp}` row on every change.

---

## DynamoDB: Analytics Aggregate Item

For per-entity view counts and time-series analytics, maintain two item types in the same partition:

- **Event items:** `SK = EVENT#{ISO_timestamp}#{uuid}` — one item per event, written at ingestion time. Include all required fields at write time: `viewer_email`, `group_slug`, `duration_seconds`, `access_method`.
- **Aggregate item:** `SK = ANALYTICS_SUMMARY` — maintained via atomic `ADD` expressions for counters and map attribute expressions for keyed values (e.g., per-viewer duration map keyed by email).

Read the aggregate item for dashboard counts — O(1). Query event items with a bounded 30-day date range for time-series charts. Never query all events to compute a total; the aggregate item exists to make that unnecessary.

Use a distinct SK prefix for the aggregate item. A `begins_with EVENT#` prefix filter on event queries cannot accidentally return the aggregate item.

Guard against a missing aggregate item on first read by treating `item not found` as a zeroed counter. The aggregate item may not exist until the first event is recorded.

**Duration accumulation.** Track active time as an event-driven accumulator, not a polling interval. Record `activeStartedAt` when the user enters active state; accumulate `now - activeStartedAt` on leaving (visibility hidden, idle timeout, unmount, pagehide). A 30-second polling interval misses all sub-30-second visits — they record zero duration.

---

## DynamoDB: Transient Fields

Store short-lived verification or workflow state — one-time codes, expiry timestamps, in-flight tokens — directly on the primary entity item. These are transient fields, not a new table.

Example pattern for a verification flow:

| Field | Description |
|---|---|
| `verification_code` | 6-digit code (use `SystemRandom`, not `random`) |
| `verification_token` | UUID request nonce (prevents cross-request replay) |
| `verification_expires_at` | ISO timestamp (short expiry window) |

Clear transient fields before completing the destructive action they guard. Clearing them inline (in the same UpdateItem that commits the action) removes the replay window.

Overwrite on each new request — callers initiate a new flow by writing fresh values, which naturally invalidates any outstanding nonce from a previous request.

---

## DynamoDB: Sibling Items and Cooldown State

Store notification cooldown state in a dedicated sibling item rather than on the primary entity row. A sibling is an item in the same partition with a distinct SK prefix:

```
PK = ENTITY#{id}
SK = META#NOTIFICATIONS   ← cooldown sibling
SK = META#SUMMARY         ← counter sibling
SK = RECORD#{id}          ← primary entity items
```

This isolates cooldown reads and writes from entity mutations, keeping update expressions simple and contention low.

For a notification batch-and-cooldown pattern: record `last_sent_at` on the cooldown sibling. On publish, check the sibling — if within the cooldown window, skip the send; otherwise send and update `last_sent_at`. This prevents burst emails on rapid successive updates.

---

## DynamoDB: Conditional Upsert and the Phantom-Row Guard

When an upsert should only update an existing item — never create one — add a ConditionExpression:

```python
await table.update_item(
    Key={"PK": pk, "SK": sk},
    UpdateExpression="SET ...",
    ConditionExpression="attribute_exists(PK)",
)
```

Without this guard, a typo in the key or a caller that races ahead of table creation silently creates a phantom row. Phantom rows are invisible until a query returns them — at which point they appear as corrupted data.

For an idempotent claim pattern (first writer wins), combine `attribute_not_exists` with `attribute_exists`:

```python
ConditionExpression=(
    "attribute_exists(#slug) AND "
    "(attribute_not_exists(claimed_by) OR claimed_by = :null)"
)
```

---

## Aurora Serverless v2

Use Aurora Serverless v2 (PostgreSQL) for permission and membership entities that require joins, counts, and complex filters across entity types. DynamoDB is the content store; Aurora is the permission store. Enforce a single-service boundary: only one module (e.g., `permission_service.py`) imports `asyncpg`.

**Driver split.** Use `asyncpg` for runtime query execution inside async handlers. Use `psycopg2-binary` for Alembic migration scripts — Alembic's sync interface requires a sync driver.

**Feature flag.** Gate Aurora initialization behind an env var (e.g., `AURORA_ENABLED=false`). The application starts without Aurora disabled; feature-flagged paths skip Aurora calls when the flag is false. This lets the backend run locally without an Aurora connection.

**Capacity.** Start at 0.5–2 ACU for low-traffic workloads. Aurora Serverless v2 scales to zero when idle, which eliminates cost during dev periods.

**Local development.** Run PostgreSQL 15 via Docker Compose for local parity. Use the same database name and user as production. Document the env var that switches the application from the Aurora endpoint to the local endpoint.

**Cross-DB orchestration.** When a feature spans both DynamoDB and Aurora, implement it in a dedicated service module — not inside either the DDB service or `permission_service`. This isolates the cross-DB orchestration pattern and keeps each store's service module focused on its own driver.

**Alembic migrations.** Maintain migrations in `alembic/versions/`. Use sequential integer prefixes (`001_`, `002_`) for readability. Also provide an idempotent raw SQL file for emergency manual application via the RDS Query Editor — Alembic may not be available in all incident recovery scenarios.

**Denormalized feed projection.** For browsable feed queries (e.g., a public listing), maintain a denormalized Aurora table that stores only the columns needed for listing: primary key, routing slug, filter columns. Apply a partial index on the most selective filter:

```sql
CREATE INDEX idx_browse_feed ON browse_index (overall_score, is_hidden)
WHERE overall_score IS NOT NULL AND is_hidden = FALSE;
```

Include routing keys in the projection so click-throughs don't require a second lookup.

---

## S3: Upload Bucket and Presigned URLs

**Bucket security defaults.** Enable all four `PublicAccessBlock` settings. Enforce TLS with a bucket policy `Deny` on `aws:SecureTransport: false`. Enable versioning on prod buckets. Set a lifecycle rule to abort incomplete multipart uploads after 7 days.

**CORS.** Configure S3 CORS to allow `PUT` and `GET` from your frontend origin(s), including `localhost:3000` for local development. Without S3 CORS, browser presigned PUT requests are rejected before reaching the bucket.

**Two-phase upload.** Issue a presigned PUT URL (step 1). Require an explicit confirm call after the client finishes uploading (step 2) to transition the record from `uploading` to `processing`. This prevents orphaned records from abandoned uploads and decouples the API from the S3 data transfer.

**Presigned URL generation.** Generate presigned URLs server-side using `aioboto3`. Set an expiry appropriate for expected upload size — large files (video) need longer windows. Pass the `ContentType` and optionally a max `ContentLength` condition to prevent content-type spoofing.

**File size enforcement.** Enforce size limits at three layers: client-side validation, backend Pydantic validation before generating the presigned URL, and a `ContentLengthRange` condition on the presigned URL itself. Any single layer is insufficient.

**Deploy bucket pattern.** Use a separate S3 bucket (`{app}-deploy-{env}`) for Lambda deployment packages. The packaging script zips each Lambda with its dependencies and uploads to this bucket. CloudFormation references the S3 object key and version — a new upload automatically triggers a Lambda function update on the next stack deployment.

---

## Lambda Functions

**Packaging.** Package each Lambda as a zip containing the handler and its dependencies. Store the zip in the deploy bucket. Reference the S3 object in CloudFormation — do not inline code in the template. Use a packaging script (`infra/scripts/package-lambdas.sh`) that zips each function directory and uploads with a versioned key.

**IAM.** Give each Lambda a dedicated role with minimum necessary permissions. `AWSLambdaBasicExecutionRole` (CloudWatch Logs) is the only managed policy needed. Add inline policies for S3, DynamoDB, and Bedrock access scoped to the exact resource ARNs:

```yaml
- Sid: BedrockInvoke
  Effect: Allow
  Action: bedrock:InvokeModel
  Resource: !Sub "arn:aws:bedrock:${AWS::Region}::foundation-model/${BedrockModelId}"
```

**Python path.** In Lambda, the handler zip is extracted to `/var/task`. Import your handler module as a flat import — do not rely on relative imports or a `src/` layout unless you set `PYTHONPATH` explicitly.

**Build vs runtime phase.** Some Lambda runtimes build the deployment package in a phase that is discarded before execution. Install all runtime dependencies into the zip, not into a build-phase layer that may not survive to execution.

**Use Lambda for compute-bound, stateless tasks.** Document conversion (DOCX/PPTX/XLSX → PDF via LibreOffice), async AI analysis triggered by Step Functions, and other batch workloads are good Lambda targets. Long-running, stateful, or latency-sensitive pipelines belong in the application server as background tasks.

---

## SES: Email Service

**Never start with `COGNITO_DEFAULT` sending.** Any Cognito UserPool that sends email to users must set `EmailConfiguration.EmailSendingAccount: DEVELOPER` and reference an SES domain identity ARN from day one. The default Cognito shared sender has a 50-email/day hard cap and a shared low-reputation IP pool — auth emails land in spam within hours of launch.

**Required Cognito email configuration:**

```yaml
EmailConfiguration:
  EmailSendingAccount: DEVELOPER
  SourceArn: !ImportValue myapp-prod-SesDomainIdentityArn
  From: "MyApp <noreply@myapp.com>"
  ConfigurationSet: !ImportValue myapp-prod-SesConfigSetName
```

**Deployment ordering is mandatory:**
1. Deploy the SES stack (domain identity + configuration set).
2. Publish DKIM CNAMEs to Route 53 manually — CloudFormation does not auto-publish these.
3. Wait for DKIM status to reach `Success` in the SES console.
4. Deploy or update the Cognito stack referencing the SES identity ARN.

Deploying Cognito before SES verification causes Cognito to fall back to the default shared sender silently. The error is not surfaced in CloudFormation — it only appears in delivered email headers.

**Sandbox mode.** SES accounts start in sandbox mode — only verified addresses receive mail. Request production access early. In the request, describe only transactional sends, confirmed opt-in recipients, and bounce/complaint handling.

**DMARC ramp.** Start at `p=none`. Collect aggregate reports for one to two weeks. Escalate to `p=quarantine` only after confirming DKIM and SPF alignment in reports. Never go straight to `p=reject` without observation.

**End-to-end verification.** After any SES or Cognito config change, trigger a real send and inspect raw email headers for `dkim=pass / spf=pass / dmarc=pass`. A passing CloudFormation deploy does not prove email deliverability.

---

## Bedrock: Invocation and Caching

**Response key.** `invoke_model` returns a `StreamingBody` at key `"body"` (lowercase). `s3.get_object` returns `"Body"` (uppercase). These are different response shapes from different services. Confusing them produces a `KeyError` that tests may not catch if mock responses are hand-written.

**Model ID for cross-region inference.** Claude 4.x models require a cross-region inference profile ID, not a base model ID. The profile ID carries a region prefix:

```
Wrong:   anthropic.claude-opus-4-6-v1
Correct: us.anthropic.claude-opus-4-6-v1
```

Use the base model ID format (e.g., `anthropic.claude-sonnet-4-20250514-v1:0`) only for models that support on-demand invocation directly. Check the Bedrock console model card for the correct inference profile ID before wiring a new model.

**Two distinct Bedrock APIs — never mix their syntax.**

The `InvokeModel` API (native Anthropic Messages format) uses `cache_control` on a typed content block:

```python
{
    "type": "text",
    "text": "...",
    "cache_control": {"type": "ephemeral"}
}
```

The Converse API uses `cachePoint`:

```python
{
    "cachePoint": {"type": "default"}
}
```

Copying a fragment from one API shape into the other causes a validation error. `InvokeModel` rejects any content block missing a `type` field.

**Caching verification.** After deploying a prompt caching change, check CloudWatch metrics for `cacheReadInputTokens`. A non-zero value confirms the cache is active. An absence indicates the cache control block is malformed or the model does not support caching at that context length.

**IAM.** Grant `bedrock:InvokeModel` scoped to the exact foundation model ARN:

```
arn:aws:bedrock:{region}::foundation-model/{model_id}
```

A wildcard on `Resource: "*"` works but violates least-privilege. The model ARN format uses `::` (no account ID) because foundation models are AWS-managed resources.

---

## CloudFormation Stack Architecture

**11-stack pattern.** Organize stacks by service boundary: deploy-bucket, cognito, dynamodb, s3, rds, ses, lambdas, step-functions, app-runner, amplify. Deploy in dependency order. Export outputs from upstream stacks and import them in downstream stacks with `!ImportValue`.

**Cross-stack output dependencies:**

| Upstream | Output | Consumed by |
|---|---|---|
| deploy-bucket | `DeployBucketName` | lambdas |
| cognito | `UserPoolId`, `UserPoolClientId`, `JwksUrl` | lambdas, app-runner, amplify |
| dynamodb | `DocumentsTableArn` | lambdas |
| s3 | `BucketArn`, `BucketName` | lambdas, app-runner |
| rds | `ClusterEndpoint`, `SecretArn` | app-runner |
| lambdas | `AnalyzeFunctionArn`, `ConvertFunctionArn` | step-functions |
| step-functions | `StateMachineArn` | app-runner |

**Parameter files.** Maintain separate parameter files per environment (`parameters/dev.json`, `parameters/prod.json`). Never hard-code account IDs, ARNs, or domain names in templates — parametrize them.

**Separate application and infrastructure pipelines.** A broken IaC template must not block an application hotfix deploy. Trigger the infrastructure pipeline only on changes to `infra/` and Lambda source paths.

---

## IAM Ordering and Naming

**Naming consistency.** IAM policy `Resource` ARNs must use the exact resource name, including separator characters. DynamoDB table names use underscores (`myapp_events`); a policy with a dash (`myapp-events`) matches nothing and silently denies all access. This is a deployment-time pitfall with no runtime warning until the first denied call.

**GSI permissions.** Grant DynamoDB permissions on both the table ARN and the index ARN:

```yaml
Resource:
  - !Ref MyTable
  - !Sub "${MyTable.Arn}/index/*"
```

Omitting the index ARN causes `AccessDeniedException` on any GSI query, even when the table-level action is permitted.

**New service types always need explicit policy additions.** Wildcard policies on an existing service type (e.g., `dynamodb:*` on `myapp_*`) do not extend to a new service type (e.g., `cognito-idp:AdminGetUser`). Enumerate each new service permission explicitly. Document it in the commit message and execution plan.

**App Runner instance role.** The instance role is the identity your application code runs as. Any AWS SDK call from a FastAPI handler uses this role. Add permissions to the instance role — not to a developer IAM user — for DynamoDB, S3, SES, Bedrock, Step Functions, MediaConvert, and Cognito admin actions.

---

## Cross-references

- `core/practices/infra-management.md` — plan-mode approval gate; net infra footprint check; deployment sequencing; post-deploy infrastructure verification.
- `core/practices/architecture-principles.md` — storage selection; companion-document pattern; analytics aggregate item; conditional upsert; source-agnostic schema design; denormalized feed projection.
- `practices/deployment-aws.md` — App Runner and Amplify deployment, health checks, env var wiring, custom domain mapping.
- `practices/security-aws.md` — Cognito JWT verification, secret management, presigned URL expiry policy.
- `practices/conventions-fullstack.md` — `aioboto3` async discipline, `build_update_expression` shared helper, Pydantic Settings pattern.
- `practices/testing-fullstack.md` — moto fixtures for DynamoDB/S3/SES, Bedrock mock discipline, integration test table setup.

---

<!-- Assembled from: docs/infrastructure.md §AWS Resources / Aurora Serverless v2 / Lambda / CI/CD / Environment Variables / Deployment Guide / Troubleshooting; docs/data-model.md §Design Philosophy / DynamoDB Tables Summary / Key Access Patterns / Analytics / Founder Updates / Investor Pool + Matches / Aurora Tables / Account Deletion Fields; docs/architecture.md §System Overview / Backend Layering; infra/templates/dynamodb.yaml §DataroomsTable / UsersTable (PAY_PER_REQUEST, PITR, GSI patterns); infra/templates/s3.yaml §UploadBucket / BucketPolicy / CorsConfiguration; infra/templates/ses.yaml §ConfigurationSet / EmailIdentity; infra/templates/lambdas.yaml §AnalyzeFunctionRole / IAM / BedrockInvoke; core/practices/infra-management.md §Infrastructure Change Approval Gate / Deployment Sequencing / CI/CD Pipeline Structure; core/practices/architecture-principles.md §Analytics in a Document Store / Conditional upsert / Document-Store Schema Patterns; ERRATA.md §ERR-003 missing DDB tables / ERR-004 Bedrock response key / ERR-005 cross-region inference profile / ERR-007 Bedrock API syntax / ERR-008 Cognito/SES email wiring / ERR-011 analytics instrumentation -->
