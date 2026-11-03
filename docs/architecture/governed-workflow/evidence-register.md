# Evidence Register (governed workflow — evidence pack, part 3/4)

Commits: R0 `dbb032e8fbb4`, R1 `b94a2af57967`, R2 `ab6bf218feb5`, R3 `4a06aef6cd75`, R4 `1c5ae50d1bd1`,
R5 `56c43c0e2c05`, R6 `f58138ebc01c`, R7 `a98f0c7de91d`, R8 `dee3765fe41c`, R9 `b33139f357b8`.

Classes:
- **CODE** (application or migration code), **CONFIG**, **DOC**, **TEST**, **MOCK** (test double), **LIVE**.
- **ABSENT**: the item was searched for and not found.

Confidence: **H**igh, **M**edium, **L**ow.

| ID | Repo | Path | Symbol / config key | Lines | Class | Conf. |
|---|---|---|---|---|---|---|
| E-01 | R1 | `app/api/v1/auth.py`; `users.py`; `roles.py` | `@router.post/get` | auth 12-27; users 14-63; roles 11-14 | CODE | H |
| E-02 | R2 | `app/api/v1/patients.py` | `router`, `prefix="/patients"` | 14-81 | CODE | H |
| E-03 | R3 | `app/api/v1/providers.py`; `specialties.py` | routes | providers 13-100; specialties 13-52 | CODE | H |
| E-04 | R4 | `app/api/v1/facilities.py`; `departments.py` | routes | facilities 16-145; departments 15-68 | CODE | H |
| E-05 | R5 | `app/api/v1/working_hours.py`; `exceptions.py`; `slots.py` | routes | wh 24-84; exc 23-93; slots 34 | CODE | H |
| E-06 | R6 | `app/api/v1/appointments.py` | `prefix="/appointments"`, 8 routes | 29-191 | CODE | H |
| E-07 | R1–R6 | `app/api/health.py` | `health`, `readiness` (`SELECT 1`) | 11-19 | CODE | H |
| E-08 | R5 | `app/clients/facility_client.py` | `HttpFacilityClient.get_facility` | 35-87 | CODE | H |
| E-09 | R6 | `app/clients/scheduling_client.py`; `app/services/appointment_service.py` | `is_available`; `_advise` | 67-100; 165-175 | CODE | H |
| E-10 | R6 | `app/services/appointment_service.py`; `app/models/outbox_event.py` | `_save`, `_event`; `OutboxEvent` | 188-230; 19-31 | CODE | H |
| E-11 | R8 | `infra/terraform/modules/sqs/main.tf`; `variables.tf`; `environments/dev/main.tf` | `aws_sqs_queue.this/dlq`, `tls_only` policy; defaults 60 s / 345600 s / 5 | 1-55; 6-21; 127-132 | CONFIG | H |
| E-12 | R8 | `environments/dev/main.tf`; `terraform.tfvars` | `module "ses"` count; `hosted_zone_name = null` | 134-136; 111 | CONFIG | H |
| E-13 | R1 | `app/models/user.py`; `migrations/versions/0001_create_users_table.py`; `app/services/seed_service.py` | `users`; `DemoAccount` | 22-25 (seed) | CODE | H |
| E-14 | R2 | `app/models/patient.py`; `app/schemas/patient.py` | `PatientSummary`, `PatientRead` | 81-105 | CODE | H |
| E-15 | R3 | `app/models/provider.py`, `specialty.py` | tables `providers`, `specialties`, `provider_specialties` | — | CODE | H |
| E-16 | R4 | `app/models/facility.py`, `department.py`, `room.py` | tables | — | CODE | H |
| E-17 | R5 | `app/models/working_hours.py`, `availability_exception.py` | tables | — | CODE | H |
| E-18 | R6 | `app/models/appointment.py`; `outbox_event.py` | tables `appointments`, `outbox_events` | 19-63 | CODE | H |
| E-19 | R6, R3 | `app/models/appointment.py`; `provider-service/app/schemas/provider.py` | UUID refs without FK | 21,49-51; 53-54 | CODE | H |
| E-20 | R8 | `environments/dev/terraform.tfvars`; `modules/rds/main.tf` | `rds_multi_az=false`, `rds_backup_retention_period=7`; `rds.force_ssl` | 99-106; 33 | CONFIG | H |
| E-21 | R6, R1, R2 | `app/models/appointment.py`; `user_repository.py`; `patient_repository.py` | `ExcludeConstraint`, `version_id_col`; `with_for_update` | 39,63; 45; 52 | CODE | H |
| E-22 | R0 | `.github/workflows/build-push.yml`; `ci.yml` | `build`, `release`; service matrix | whole files | CONFIG | H |
| E-23 | R0 | live Argo CD (`kubectl -n argocd get applications`) | `targetRevision=main`, 11 apps Synced/Healthy | — | LIVE | H |
| E-24 | R7/R0 | `deploy/helm/*-service/values.yaml`; `dev-values.yaml`; R8 `terraform.tfvars` | `replicaCount` 2 / 1; `eks_node_capacity_type=SPOT` | values 15; dev 8; tfvars 23-31 | CONFIG | H |
| E-25 | R7 | `deploy/helm/patient-service/values.yaml` | `resources` | 36-42 | CONFIG | H |
| E-26 | R7 | `deploy/helm/microservice/templates/_migrationjob.tpl` | `argocd.argoproj.io/hook: Sync`, `sync-wave: "-1"` | 10-12 | CONFIG | H |
| E-27 | R8; R7 | `terraform.tfvars` `services[*].path_patterns`; `deploy/argocd/install-values.yaml` | ALB rules; `server.service.type: LoadBalancer` | 37-92; 24 | CONFIG + LIVE | H |
| E-28 | R8 | `environments/clinical-platform/*` | root module | — | CONFIG; LIVE (absent) | H |
| E-29 | R1; R2–R6 | `app/core/tokens.py` | `issue_access_token`; `verify` | R1 14,40-61; others 35-63 | CODE | H |
| E-30 | R2 (pattern in all) | `app/domain/roles.py`; `app/api/deps.py` | `ROLE_PERMISSIONS`; `require_permission` | 18-35; 61-67 | CODE | H |
| E-31 | R8; R7 | `modules/secrets/main.tf`; `_externalsecret.tpl` | `jwt_signing_key`; `JWT_SECRET_KEY`, `JWT_KEY_ID` | 34-49; 44-51 | CONFIG | H |
| E-32 | R2 (all) | `app/core/config.py` | `_PLACEHOLDER_MARKERS`, `MIN_SECRET_BYTES_BY_ALGORITHM` | 10-27, 64 | CODE | H |
| E-33 | R7; R0 | `_externalsecret.tpl`; `ci.yml` `secret-scan` | ESO; gitleaks | 1-51; job `secret-scan` | CONFIG | H |
| E-34 | R8 | `terraform.tfvars`; `modules/security-groups/main.tf`; `modules/waf/main.tf` | `eks_endpoint_public_access_cidrs`; ingress rules; `rate-limit-per-ip`, managed rules | 24; 15-30; 11-45 | CONFIG | H |
| E-35 | R8 | `modules/alb/main.tf` | HTTP listener (no ACM without a domain) | — | CONFIG | M |
| E-36 | R1–R6; R7 | `Dockerfile`; `_deployment.tpl` | `USER 10001:10001`; `securityContext` | 21; 38-39, 75-76 | CONFIG | H |
| E-37 | R5, R6 | `app/core/config.py` | `facility_timeout_seconds`, `scheduling_timeout_seconds` = 3.0 | 54; 61 | CODE | H |
| E-38 | R1–R6 | `app/**` (search: retry, backoff, circuit, idempot, fallback, tenacity) | only messages at `appointment_service.py:22`, `patient_service.py:71`, `provider_service.py:139`; seed idempotency `seed_service.py:37` | — | ABSENT (code) | H |
| E-39 | R7 | `deploy/helm/appointment-service/dev-values.yaml` | `SCHEDULING_ADVICE_ENABLED: "true"`, `APPOINTMENTS_ENABLED: "true"` | — | CONFIG | H |
| E-40 | R5 | `tests/conftest.py`; `tests/test_facility_client.py` | `FakeFacilityClient`; `MockTransport` | 84 | MOCK | H |
| E-41 | R6 | `tests/conftest.py`; `tests/test_scheduling_advice.py` | `FakeSchedulingClient`; `HttpSchedulingClient` with a handler | 88; 22-23 | MOCK | H |
| E-42 | all | repo-wide search (openapi-generator, pact, schemathesis, locust, k6, pytest-benchmark) | — | — | ABSENT | H |
| E-43 | R1–R6 | `app/db/session.py` | `create_engine(..., pool_pre_ping=True)` | 10 | CODE | H |
| E-44 | R7 | `deploy/argocd/apps/values.yaml` | `syncPolicy` automated, `retry.limit: 10` | 14-26 | CONFIG | H |
| E-45 | R1–R6 | `app/core/logging.py`; `app/api/middleware.py` | JSON logging; `X-Request-ID` | 1; 18-43 | CODE | H |
| E-46 | R7 | `deploy/datadog/values.yaml` | `apm`, `instrumentation` (SSI) | 17-22 | CONFIG | H |
| E-47 | R8 | `modules/datadog-observability/slos.tf`; `variables.tf` | `availability_target=99.5`, `latency_threshold_ms=500`, `latency_target=99.5` | 20-61; 24-36 | CONFIG | H |
| E-48 | R8 | `modules/datadog-observability/monitors.tf` | 7 `datadog_monitor` resources | 15-95 | CONFIG | H |
| E-49 | R7; R9; R6 | `deploy/README.md` "Rollout and rollback"; `docs/workflow/governed-sdlc.md` §5; `README.md` rollback | — | — | DOC | H |
| E-50 | R1–R6 | `tests/test_*.py` | 415 test functions / 51 files | — | TEST | H |
| E-51 | R0 | `.github/workflows/ci.yml` | `Tests (PostgreSQL)`, `Alembic round trip` | — | CONFIG | H |
| E-52 | R6 | `tests/test_concurrency_postgres.py` | PostgreSQL-only concurrency tests | — | TEST | H |
| E-53 | R8 | `environments/dev/tests/plan.tftest.hcl` | `mock_provider "aws"` | 1-80 | TEST (mocked) | H |
| E-54 | — | smoke script run through the ALB (not in a repository) | 13 steps PASS on 2026-11-03 | — | LIVE | M |
