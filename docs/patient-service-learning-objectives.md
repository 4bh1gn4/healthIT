# Patient Service — Learning Objectives

These objectives guide contributors through the core competencies needed to design, build, secure, deploy, and operate the Patient Service (Phase 1: demographics) locally and in the cloud.

## Architecture & Domain
- Explain the service purpose, scope, and Phase 1 boundaries (demographics only).
- Model a Patient domain with clear ownership of identifiers, soft-delete, and versioning.
- Describe trade-offs of stateless API + Postgres persistence and when to add caching.

## API Design & Behavior
- Produce and maintain an OpenAPI v3 spec for versioned endpoints (`/api/v1`).
- Implement CRUD for Patients with idempotent create (`Idempotency-Key`).
- Enforce optimistic concurrency with `ETag`/`If-Match` on updates; return 412 on mismatch.
- Implement soft-delete and restore semantics with predictable status codes.
- Provide consistent error shapes `{ code, message, details? }` and pagination.

## Data Modeling & Storage
- Design a Postgres schema with appropriate types, constraints, and indexes.
- Implement nullable-unique patterns (e.g., email/phone) and soft-delete filtering.
- Write and apply versioned DB migrations; verify forward/backward compatibility.
- Use transactions for create/update flows and understand isolation implications.

## Validation & Normalization
- Validate names, DOB (past, ≤120 years), email, E.164 phone, and enums.
- Normalize emails (lowercase) and phones (E.164) for uniqueness.
- Protect logs: redact PII/PHI and avoid payload dumping.

## Security & Compliance Foundations
- Secure the API with API key or OIDC bearer tokens and minimal scopes.
- Store and access secrets using environment variables locally and a cloud secret manager.
- Enforce encryption in transit (TLS) and at rest (DB, volumes, images, secrets/KMS).
- Record auditable changes (actor, action, timestamp, diff) without leaking PHI.

## Observability
- Emit structured JSON logs with request/trace IDs and key dimensions.
- Expose health (`/healthz`) and readiness (`/readyz`) probes with useful signals.
- Collect metrics (RPS, latency p50/p95/p99, error rate) and traces around DB calls.
- Configure basic alerts (error rate, latency, task health, DB CPU/conn thresholds).

## Local Developer Experience
- Run the stack via Docker Compose (API, Postgres, migrations) using `.env`.
- Seed data, run migrations, and execute tests locally with Make targets.
- Troubleshoot common local issues (port conflicts, DB migrations, env mismatch).

## Infrastructure as Code (Terraform)
- Author reusable modules for network, database, app, secrets, and observability.
- Configure remote state + locking; plan, apply, and destroy safely in `envs/`.
- Provision VPC, ECS Fargate service, ALB, RDS Postgres, IAM, Secrets Manager, and KMS.
- Output endpoints and ARNs; wire ECS task to secrets and DB securely.

## CI/CD & Supply Chain
- Build a pipeline that lints, tests, builds/pushes images, and runs Terraform plan.
- Implement deployment strategy (rolling/blue-green) with health checks.
- Generate SBOM and run image/IaC scanning; gate merges on critical findings.

## Testing Strategy
- Write unit tests for validation, idempotency, and ETag logic.
- Add integration tests against ephemeral Postgres and API contract tests.
- Execute load smoke tests (e.g., k6) to validate p95 targets.

## Performance & Reliability
- Meet SLOs: p95 GET ≤ 50 ms; POST/PUT ≤ 200 ms (in-region baseline).
- Size DB connections and indexes for target throughput and data growth.
- Apply runbooks for rollback, secret rotation, and DB restore drills.

## Cloud Deployment & Operations
- Deploy to a dev environment; validate HTTPS via ALB and readiness/health probes.
- Inspect logs/metrics to confirm request flow and error handling.
- Verify RDS encryption, private networking, and minimal IAM privileges.

## Interoperability (Forward-Looking)
- Map demographics to FHIR Patient fields; identify gaps for future phases.
- Plan the path to OIDC/OAuth2 scopes and multi-tenant authorization.

---

### Evidence of Mastery (Examples)
- Given two identical POSTs with the same `Idempotency-Key`, returns the same 201 response.
- Updating a patient without a matching `If-Match` returns 412 and does not modify the record.
- Terraform apply creates VPC, ECS service, RDS (encrypted), ALB (HTTPS), and Secrets; service becomes healthy.
- Logs contain request IDs and timing but no PII/PHI; alerts trigger on high 5xx/error rate.
