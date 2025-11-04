# Patient Service Specification (Phase 1)

## Summary
- Build a small Patient Service to collect and manage basic patient demographics.
- Phase 1 delivers create/read/update of demographics, runs locally (Docker) and in cloud (Terraform IaC).
- Security-first, HIPAA-aware foundation; API versioned, testable, observable, and portable.

## Scope (Phase 1)
- CRUD for Patient demographics (single system of record).
- Idempotent create, optimistic update, soft-delete.
- Local dev via Docker Compose; cloud infra via Terraform.
- Minimal auth (API key) with clear path to OAuth2/OIDC.
- Basic logging/metrics/tracing; health and readiness endpoints.

## Non-Goals (Phase 1)
- Scheduling, clinical data, claims, or billing.
- Advanced patient matching/deduplication.
- Patient portal/UX.
- Cross-organization interoperability (e.g., FHIR endpoints) beyond basic mapping.

## Functional Requirements
- Create patient with demographics; return stable `patient_id` (UUIDv4).
- Read patient by `patient_id`.
- Update demographics with optimistic concurrency (If-Match ETag).
- List patients with filters (family_name, dob, email, phone).
- Soft delete and restore.
- Idempotency on POST via `Idempotency-Key`.
- Minimal search pagination (limit, cursor or offset).

## Non-Functional Requirements
- Availability: 99.9% cloud target for Phase 1.
- Latency: p95 GET ≤ 50 ms, POST/PUT ≤ 200 ms (in-region).
- Throughput: 50 rps baseline; horizontally scalable.
- Data durability: automated backups, PITR where supported.
- Observability: structured logs, metrics, traces; health/readiness endpoints.
- Compliance-ready: encryption in transit/at rest; audit logs for changes.

## Data Model
### Patient
- `patient_id`: UUIDv4 (PK)
- `given_name`: string (1–100)
- `middle_name`: string? (0–100)
- `family_name`: string (1–100)
- `date_of_birth`: date (YYYY-MM-DD)
- `sex_at_birth`: enum [female, male, intersex, unknown, undisclosed]
- `gender_identity`: enum [woman, man, nonbinary, other, undisclosed] + free-text when other
- `email`: string? (RFC 5322; unique nullable)
- `phone_e164`: string? (E.164; unique nullable)
- `address`:
  - `line1`: string (1–200)
  - `line2`: string?
  - `city`: string
  - `state_province`: string
  - `postal_code`: string
  - `country`: string (ISO 3166-1 alpha-2)
- `preferred_language`: string? (IETF BCP 47)
- `consent_to_contact`: boolean (default false)
- `created_at`/`updated_at`: timestamptz
- `version`: int (optimistic locking)
- `is_deleted`: boolean (soft delete)

### Idempotency
- `key`: string (hash of header)
- `request_fingerprint`: string
- `response_code`/`body_hash`
- `expires_at`: timestamptz

### AuditLog
- `audit_id`, `actor`, `action` [create, update, delete, restore], `patient_id`, `diff`, `occurred_at`

## Validation Rules
- Names: trim, collapse whitespace; Unicode allowed; disallow digits in names.
- DOB: past date; age ≤ 120 years.
- Email: format check; normalized lower-case for uniqueness.
- Phone: E.164 format; store canonical form.
- Address: country required if postal_code provided; basic per-country format checks where feasible.
- Enums: strict; provide “other” + free-text where applicable.
- PII/PHI in logs: never log full payloads; redact PII fields.

## API (v1)
- POST `/api/v1/patients`
  - Headers: `Content-Type: application/json`, `Idempotency-Key` (recommended), `Authorization: Bearer <token>` or `x-api-key`
  - Body: demographics fields (no `patient_id`)
  - 201 with body `{ patient_id, ... }`, headers: `ETag`
  - 409 on conflict (e.g., optimistic concurrency/idempotency mismatch)
- GET `/api/v1/patients/{patient_id}`
  - 200 with patient; headers: `ETag`
  - 404 if not found
- PUT `/api/v1/patients/{patient_id}`
  - Headers: `If-Match: <ETag>` required
  - Body: full resource or PATCH-style? Phase 1: full replace (PUT)
  - 200 with updated resource; new `ETag`; 412 on ETag mismatch
- DELETE `/api/v1/patients/{patient_id}`
  - Soft delete; 204
- POST `/api/v1/patients/{patient_id}:restore`
  - 200 with restored resource
- GET `/api/v1/patients?family_name=&dob=&email=&phone=&limit=&cursor=`
  - 200 with list, `next_cursor`
- GET `/healthz` (liveness), `/readyz` (readiness)
- Errors: JSON `{ code, message, details? }` with stable error codes.

## Architecture
- Service: stateless HTTP API.
- Persistence: PostgreSQL (transactions, JSONB for address optionality).
- Caching: none in Phase 1; rely on DB indexes; optional read-through cache later.
- Idempotency middleware and audit logging.
- Config via env vars; secrets from cloud secret manager in cloud, `.env` locally.

## Local (Developer)
- Docker Compose services: `api`, `postgres`, `migrations`, optional `pgadmin`.
- `make up`, `make down`, `make test` convenience targets.
- Seed/migration scripts; sample `.env.example`.

## Cloud (Reference: AWS; provider-agnostic structure)
- Compute: ECS Fargate (or Cloud Run/Azure Container Apps analogs).
- Network: VPC, private subnets, NAT; ALB for HTTPS.
- DB: Amazon RDS Postgres with Multi-AZ, automated backups, PITR.
- Secrets: AWS Secrets Manager; KMS CMKs for encryption at rest.
- Logs/Metrics: CloudWatch logs, ALB/NLB access logs to S3; X-Ray tracing optional.
- DNS/TLS: Route 53, ACM certificates; WAF optional.

## Terraform Structure
- Root: `infra/terraform/`
  - `modules/`
    - `network` (VPC, subnets, NAT, gateways)
    - `database` (RDS Postgres, subnet groups, parameter groups)
    - `app` (ECR repo, ECS cluster/service/task, ALB, target groups, IAM)
    - `observability` (log groups, metrics alarms, dashboards)
    - `secrets` (Secrets Manager, KMS keys, IAM policies)
  - `envs/`
    - `dev`, `staging`, `prod` (each with `main.tf`, `providers.tf`, `variables.tf`, `backend.tf`)
- Patterns
  - Remote state (S3 + DynamoDB lock).
  - Per-env var files; minimal duplication.
  - Outputs for endpoints, DB endpoints, secret ARNs.
  - `terraform validate`, `plan`, `apply` wired in CI.

## Security & Compliance
- AuthN/AuthZ
  - Phase 1: `x-api-key` or OIDC bearer tokens with minimal scopes (`patient:read`, `patient:write`).
  - Enforce least privilege IAM for tasks, DB, Secrets Manager.
- Crypto
  - TLS 1.2+ everywhere; HSTS at ALB; no HTTP downgrade.
  - Encryption at rest: RDS, ECR, EBS, S3, Secrets via KMS.
- PHI/PII Handling
  - Minimize logging; use field-level redaction.
  - Audit trail for changes (actor, timestamp, diff).
- Backups/DR
  - Automated RDS snapshots; restore runbooks tested in non-prod.
- Access
  - Break-glass access process; all prod access audited.
- HIPAA
  - BAA with cloud provider; retain logs 6+ years policy-ready (configurable).

## Observability
- Logs: JSON; include request_id, trace_id, actor, route, status, duration; no PHI.
- Metrics: request count, latency (p50/p95/p99), error rate, 4xx/5xx, DB connections, queue depth (if any).
- Traces: propagate W3C Trace Context; spans around DB calls.
- Alerts: on high error rate, p95 latency breach, unhealthy tasks, RDS CPU/conn thresholds.

## Operations
- Migrations: versioned (e.g., Alembic/Flyway); run on deploy or pre-deploy job.
- Runbooks: deploy, rollback, rotate secrets, restore DB, investigate high latency.
- Config flags: kill-switch for writes in emergencies.

## Testing & QA
- Unit tests for validation, idempotency, ETag handling.
- Integration tests with ephemeral Postgres in CI.
- Contract tests for API (e.g., Dredd/Postman/Newman).
- Load smoke test (k6) at target RPS; assertions on p95 latency.
- Terraform checks: `fmt`, `validate`, `tflint`, `checkov` (if allowed).

## CI/CD
- On PR: lint, unit/integration tests, Docker build, Terraform `plan` (non-prod).
- On main: tag image, push to ECR, run migrations, deploy to dev; manual approvals to prod.
- Blue/green or rolling ECS deploys; health checks via `/readyz`.
- SBOM and image scanning (Grype/Trivy).

## Deliverables
- API spec: `openapi.yaml` v3.0 with schemas, examples, error codes.
- Service: codebase with `Dockerfile`, `docker-compose.yml`, `Makefile`, migrations, README.
- Terraform: `infra/terraform/` modules + `envs/*`.
- Runbooks and operational docs.
- Test suites and CI pipelines.

## Acceptance Criteria
- Local: `docker compose up` exposes API; `POST /api/v1/patients` returns 201; `GET` returns 200; `PUT` respects `If-Match`; deletion is soft; basic search works.
- Cloud: Terraform `apply` creates VPC, RDS, ECS, ALB; deploy succeeds; HTTPS endpoint healthy; DB encrypted; secrets not in logs or env files.
- Observability: Logs structured; metrics exported; basic alert rule in place.
- Security: TLS enforced; API requires auth; PHI not logged; RDS not publicly accessible.

## Reference Implementation Assumptions (confirm or adjust)
- Language/Framework: Python + FastAPI (or Go + Fiber/Gin; or Node + NestJS). Default: FastAPI.
- DB: PostgreSQL 14+.
- Cloud: AWS ECS Fargate + RDS Postgres.
- Auth: API key in Phase 1; OIDC later.
- Container registry: ECR.

## Next Steps
- Confirm stack choices (language, cloud, auth).
- Draft initial `openapi.yaml`, Terraform module skeleton, and local `docker-compose.yml`.

## Open Questions
- Preferred cloud provider(s)?
- Preferred language/runtime?
- Any FHIR conformance requirements in Phase 1?
- Data residency, retention, and deletion policies?
- Expected peak RPS and region(s)?
- External identity provider for OIDC?

