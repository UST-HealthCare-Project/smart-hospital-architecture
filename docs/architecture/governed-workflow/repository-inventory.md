# Repository Inventory (governed workflow — current-state evidence pack, part 1/4)

Inspected on 2026-11-03 by read-only static analysis plus a read-only live check. No application file was changed.
This file covers **step A** (repository identity, branch, commit, access) and **step B** (component inventory).

**Evidence classes:**
- **CODE**: executable source or migrations.
- **CONFIG**: Helm, Terraform, Argo CD, Docker or CI.
- **DOC**: README, rules or markdown.
- **TEST**: test code.
- **MOCK**: a test double.
- **LIVE**: observed in a running system.

**Confidence:**
- **High**: read directly.
- **Medium**: inferred from several facts.
- **Low**: single indirect source.

## A. Repository identity, branch, commit and access

| # | Repository | Visibility | Default branch | Commit inspected (`main`) | Other branches | Access used | Notes |
|---|---|---|---|---|---|---|---|
| R1 | `UST-HealthCare-Project/identity-service` | public | main | `b94a2af57967` | `devin/1793640500-cicd-rewire` (unmerged CI) | org token (read/write); local clone `~/split/identity-service` | History split from monorepo `services/identity-service` |
| R2 | `UST-HealthCare-Project/patient-service` | public | main | `ab6bf218feb5` | same | same | — |
| R3 | `UST-HealthCare-Project/provider-service` | public | main | `4a06aef6cd75` | same | same | — |
| R4 | `UST-HealthCare-Project/facility-service` | public | main | `1c5ae50d1bd1` | same | same | — |
| R5 | `UST-HealthCare-Project/scheduling-service` | public | main | `56c43c0e2c05` | same | same | — |
| R6 | `UST-HealthCare-Project/appointment-service` | public | main | `f58138ebc01c` | same | same | — |
| R7 | `UST-HealthCare-Project/smart-hospital-gitops` | public | main | `a98f0c7de91d` | same | same | `deploy/` tree; **not** the live Argo CD source |
| R8 | `UST-HealthCare-Project/smart-hospital-infra` | public | main | `dee3765fe41c` | same | same | `infra/terraform/` tree |
| R9 | `UST-HealthCare-Project/smart-hospital-architecture` | public | main | `b33139f357b8` | 7 unmerged docs branches + CI branch | same | `main` has no `docs/architecture/` yet |
| R0 | `Msocial123/UST-HealthCare-Project` (monorepo) | public | main | `dbb032e8fbb4` | — | Devin GitHub App (read/write) | **Live deployment source**: Argo CD follows its `main` (LIVE, High) |

Access facts:
- On `main`, none of the org repos has a `.github/workflows` file. The CI and build workflows exist only on the
  unmerged branch `devin/1793640500-cicd-rewire` (CONFIG, High).
- The monorepo's `.github/workflows/ci.yml` and `build-push.yml` are the active CI/CD (CONFIG, High).
- **Drift check:** the service code in R1–R6 matches the monorepo tree for `services/<svc>` at split commit
  `63bf585` (tree-hash equality, High). Monorepo commits after the split (`b5128bb`, `dbb032e`) changed only
  `deploy/helm/db-bootstrap/dev-values.yaml`, so **R7's `deploy/helm/db-bootstrap/dev-values.yaml` is stale against the
  monorepo** (High).

## B. Service and component inventory

| # | Component | Repository | Responsibility (from code/docs) | Entry point | Tables owned | Migrations | Port | App `.py` files | Test files / test functions | Class |
|---|---|---|---|---|---|---|---|---|---|---|
| C1 | identity-service | R1 | Staff users, Argon2id password hashes, JWT issuance, roles | `app/main.py` (FastAPI factory); `app/cli.py` (`seed-demo-users`) | `users` | `0001_create_users_table.py` | 8001 (`Dockerfile` EXPOSE) | 37 | 10 / 78 | CODE High |
| C2 | patient-service | R2 | Patient registration, demographics, lookup | `app/main.py` | `patients` | `0001_create_patients_table.py` | 8002 | 31 | 6 / 53 | CODE High |
| C3 | provider-service | R3 | Provider profiles, specialties, active status | `app/main.py` | `providers`, `specialties`, `provider_specialties` | `0001_create_provider_tables.py` | 8003 | 35 | 7 / 65 | CODE High |
| C4 | facility-service | R4 | Facilities, departments, rooms, timezone | `app/main.py` | `facilities`, `departments`, `rooms` | `0001_create_facility_tables.py` | 8004 | 43 | 9 / 65 | CODE High |
| C5 | scheduling-service | R5 | Working hours, availability exceptions, advisory slots | `app/main.py` | `working_hours`, `availability_exceptions` | `0001_create_scheduling_tables.py` | 8005 | 44 | 10 / 88 | CODE High |
| C6 | appointment-service | R6 | Booking lifecycle, concurrency-safe booking, transactional outbox | `app/main.py` | `appointments`, `outbox_events` | `0001_create_appointment_tables.py` | 8006 | 36 | 9 / 66 | CODE High |
| C7 | encounter-service | — | Planned Phase 7 (visit records) | — | — | — | (NodePort 30087 reserved) | 0 | 0 | DOC (`AGENTS.md`) + CONFIG (tfvars); **no code** |
| C8 | notification-service | — | Planned Phase 8 (notification requests, delivery status) | — | — | — | (NodePort 30088 reserved) | 0 | 0 | DOC + CONFIG; **no code** |
| C9 | `microservice` Helm library chart | R7 / R0 | Deployment, Service, ServiceAccount, ConfigMap, ExternalSecret, migration Job | `deploy/helm/microservice/templates/*.tpl` | — | — | — | — | — | CONFIG High |
| C10 | db-bootstrap Job | R7 / R0 | Creates per-service PG role + DB using the RDS master secret | `deploy/helm/db-bootstrap/templates/job.yaml` | — | — | — | — | — | CONFIG High |
| C11 | Argo CD app of apps | R7 / R0 | Root app + ApplicationSet over `deploy/helm/*-service`, ESO, db-bootstrap, Datadog | `deploy/argocd/root-app.yaml`, `deploy/argocd/apps/` | — | — | — | — | — | CONFIG High |
| C12 | Terraform `dev` root | R8 / R0 | VPC, EKS, RDS, ECR, Secrets, IRSA, GitHub OIDC, ALB, WAF, SQS, SES (conditional), S3, Datadog secrets | `infra/terraform/environments/dev/main.tf` | — | — | — | — | `tests/plan.tftest.hcl` (mocked) | CONFIG High |
| C13 | Terraform `dev-observability` root | R8 / R0 | Datadog SLOs, monitors, dashboard | `environments/dev-observability/main.tf` (`module "observability"`) | — | — | — | — | — | CONFIG High |
| C14 | Terraform `clinical-platform` root | R8 / R0 | A second, separate AWS foundation | `environments/clinical-platform/` | — | — | — | — | — | CONFIG High; **LIVE: not deployed** |
| C15 | Governance docs | R9 | Rules, governed SDLC, playbook, Care Points test design | `AGENTS.md`, `Smart_Hospital_Repository_Rules.md`, `docs/workflow/*` | — | — | — | — | — | DOC High |

Pinned runtime stack (identical in all six services, CODE High):
- `fastapi==0.143.1`, `uvicorn[standard]==0.54.0`, `pydantic==2.14.1`, `pydantic-settings==2.16.0`;
- `SQLAlchemy==2.1.4`, `alembic==1.20.0`, `psycopg[binary]==3.3.6`, `PyJWT==2.15.1`;
- plus `httpx==0.28.1` in C5 and C6.
- Base image `python:3.12-slim`, running as `USER 10001:10001` (`Dockerfile:21` in each service).
