# Implementation Notes — Patient Service (Phase 1)

## Decision Summary
- Primary recommendation (cheapest over 12 months):
  - Cloud: AWS
  - Compute: API Gateway + AWS Lambda (zip deploy)
  - Database: Amazon RDS for PostgreSQL (Free Tier, single‑AZ `db.t3.micro`)
  - Secrets: Lambda environment variables for prototype (upgrade later)
- Alternative (very low cost with immediate credits or on Azure):
  - Cloud: Azure (consumption-based compute) + external free Postgres
  - Compute: Azure Container Apps (Consumption) or Azure Functions (Consumption)
  - Database: Neon or Supabase (free Postgres tier)

## Why This Choice
- Free credits/tiers:
  - AWS offers a 12‑month Free Tier that includes RDS for PostgreSQL at micro size. This keeps your DB cost near $0 for the first year if within limits.
  - Azure provides $200 credit for ~30 days and some always‑free compute, but Azure Database for PostgreSQL doesn’t have a permanent free tier. Using an external free Postgres (Neon/Supabase) on Azure avoids DB cost but places DB outside Azure.
- Minimal ongoing cost for your usage:
  - Serverless compute (Lambda or Azure Consumption) costs ~$0 at your low traffic.
  - The largest cost risk in AWS is networking (ALB, NAT Gateway). The plan below avoids both.

## Language & Framework
- Suggest Python + FastAPI for fastest iteration and simpler cold-starts on serverless.
- Java + Spring Boot is fine but typically has higher cold-start and memory overhead on serverless. If you pick Java, consider AWS Lambda SnapStart or Azure’s dedicated plans later.

## Local Development
- Database: PostgreSQL (v14+), free via Docker.
- Run with Docker Compose: API + Postgres + migrations. Keep `.env` for local secrets.
- Migrations: Alembic (Python) or Flyway/Liquibase (Java).

## AWS Low‑Cost Architecture (Primary)
- API: AWS API Gateway (HTTP API) → AWS Lambda (zip package)
- VPC: Lambda attached to a VPC with private subnets to reach RDS (no NAT Gateway).
- Database: RDS PostgreSQL single‑AZ `db.t3.micro` (Free Tier), 20GB gp2/gp3 storage.
- Secrets: Start with Lambda environment variables (non‑prod). Later move to AWS Secrets Manager or SSM Parameter Store.
- Observability: CloudWatch logs. Basic metrics/alarms (error rate, latency) are inexpensive.
- Notes to avoid surprise charges:
  - Do NOT use an Application Load Balancer (adds ~$15–$20/month).
  - Avoid NAT Gateway (adds ~$30+/month). Lambda in VPC can access RDS without NAT. Keep external calls minimal or run outside VPC for tasks needing internet.
  - Prefer zip deployment over container images to avoid ECR/egress costs/complexity initially.

## Azure Low‑Cost Architecture (Alternative)
- Option A: Azure Container Apps (Consumption) running FastAPI in a container.
- Option B: Azure Functions (Consumption) HTTP trigger wrapping FastAPI (ASGI adapter).
- Database: Neon or Supabase free Postgres tier (external to Azure).
- Networking: No load balancer needed; managed ingress included. Outbound internet is included; no NAT Gateway required.
- Secrets: Environment variables for prototype; optionally Azure Key Vault (adds small cost).

## Regions (Cost Considerations)
- AWS: `us-east-1` is commonly the least expensive general‑purpose region.
- Azure: `eastus` is typically among the cheaper regions.
- For your prototype, pick these defaults unless data residency requires otherwise.

## What We’ll Build First (once approved; no implementation yet)
- Local: `docker-compose.yml` with Postgres and the API skeleton, migrations, and a `.env.example`.
- AWS path: Terraform to provision VPC (no NAT), RDS Postgres (Free Tier), Lambda, API Gateway, and minimal IAM.
- Azure path (if chosen): Terraform/Bicep to provision Container Apps environment or Functions app; DB credentials for Neon/Supabase.

## Trade‑offs & Future Upgrades
- Secrets: Start with env vars; upgrade to Secrets Manager/Key Vault with IAM later.
- Auth: Start with `x-api-key`; move to OIDC when needed.
- Observability: Begin with logs; add metrics/tracing once traffic grows.
- HA/DR: Single‑AZ DB to start; upgrade to Multi‑AZ and backups/retention policies when moving beyond prototype.

## Your Confirmations Needed
- Pick cloud track for the first deployment: AWS (RDS free) or Azure (Container Apps/Functions + Neon/Supabase).
- Choose language runtime: Python + FastAPI or Java + Spring Boot.
- Confirm default regions: AWS `us-east-1` or Azure `eastus`.
- Are you okay starting with env vars for DB creds in cloud (prototype only)?

Once you confirm, I’ll scaffold the project and infra accordingly without incurring surprise costs.
