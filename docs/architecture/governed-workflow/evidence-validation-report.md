# Evidence Validation Report (independent review of the current-state evidence pack)

| Field | Value |
|---|---|
| Reviewed | `repository-inventory.md`, `current-state-architecture.md`, `evidence-register.md`, `discovery-gaps.md` (branch `devin/1793725037-current-state-evidence`), plus cross-document claims in the earlier architecture pack |
| Method | Re-opened every cited file at the commits in the register. Checked caller/callee wiring, whether config reaches the runtime path, and live state (read-only `kubectl` / AWS CLI) on 2026-11-03 |
| Original findings | **Not edited.** Corrections are listed here for the author to apply |

**Statuses:**
- **Verified:** the file and symbol exist and support the claim.
- **Partially verified:** true, but imprecise or incomplete.
- **Not verified:** the evidence is insufficient.
- **Contradicted:** the evidence disagrees.

## 1. Claim-by-claim results

| # | Claim (evidence ID) | Re-check performed | Status | Correction / additional evidence needed |
|---|---|---|---|---|
| V-01 | Route counts per service (E-01–E-06) | Counted `@router.<verb>(`: identity 2+5+1, patient 5, provider 6+3, facility 8+4, scheduling 4+4+1, appointment 8 | **Verified** | — |
| V-02 | `/health` + `/health/ready` (`SELECT 1`) identical in all six (E-07) | md5 of `app/api/health.py` identical (`adb8b965`) ×6 | **Verified** | — |
| V-03 | scheduling → facility client is on the runtime path (E-08) | `scheduling-service:app/main.py:15,30` always builds `HttpFacilityClient(settings.facility_service_url, settings.facility_timeout_seconds)`; `FACILITY_SERVICE_URL` and `FACILITY_TIMEOUT_SECONDS: "3"` set in `deploy/helm/scheduling-service/values.yaml:51-52` | **Verified** | — |
| V-04 | appointment → scheduling client is optional and flag-controlled (E-09) | `appointment-service:app/main.py:32-35,62` builds the client only when `scheduling_advice_enabled`; defaults false (`app/core/config.py:57`); **live** ConfigMap `appointment-service` has `SCHEDULING_ADVICE_ENABLED=true`, `APPOINTMENTS_ENABLED=true`, `SCHEDULING_TIMEOUT_SECONDS=3` | **Verified** (environment-specific) | State in the pack that the behaviour differs by environment: with code defaults, booking endpoints return 503 (flag off) and no advice call is made |
| V-05 | Booking fails closed when scheduling is unavailable (E-09; dependency map G-2) | Client maps non-2xx/transport errors to `ServiceUnavailableError` (`scheduling_client.py:82-99`); `_advise` doesn't catch it (`appointment_service.py:165-175`); tests exist in `tests/test_scheduling_advice.py` (handlers via `HttpSchedulingClient`) | **Verified** (code + tests) | Update the dependency map G-2 from "Strongly indicated, no test" to Verified, citing `tests/test_scheduling_advice.py` |
| V-06 | Outbox rows are never published (E-10) | The only write of `published_at` is `=None` (`appointment_service.py:229`); no other reference besides the model and index | **Verified** | — |
| V-07 | SQS defaults 60 s / 345600 s / 5 (E-11) | `modules/sqs/variables.tf` defaults 60, 345600, 5; the module call passes only `name` (`dev/main.tf:127-132`) | **Verified** | **Live:** 2 queues exist now (the stack was rebuilt). The earlier ARB claim C-1 "not deployed" is **time-dependent**; record the observation date with every live claim |
| V-08 | SES not created without a domain (E-12) | `count = var.enable_ses && local.use_custom_domain` (`dev/main.tf:136`); live: 0 SES identities | **Verified** | The discovery report §6 ("SES domain identity … provisioned") is **Contradicted**; correct it |
| V-09 | RDS single-AZ, 7-day backups, deletion protection (E-20) | `dev/terraform.tfvars:99-106` | **Verified** | — |
| V-10 | Concurrency control via `with_for_update` in identity and patient (E-21) | identity `user_repository.py:40-48` locks **active ADMIN users** (last-admin protection), not general rows; patient `patient_repository.py:46-54` locks on `get_by_id(for_update=True)` | **Partially verified** | State the purpose of each lock; only appointment has booking-level concurrency control (GiST + `version_id_col`, `appointment.py:39,63`) |
| V-11 | `replicaCount` 2 base / 1 dev; no HPA/PDB (E-24) | values.yaml:15 = 2; dev-values.yaml:8 = 1; no HorizontalPodAutoscaler or PodDisruptionBudget anywhere in `deploy/` | **Verified** | Add "no HPA, no PDB" as an explicit finding |
| V-12 | Container hardening (E-36) | `_helpers.tpl:26-40`: `runAsNonRoot`, `runAsUser 10001`, `seccompProfile RuntimeDefault`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` | **Verified** (stronger than stated) | Cite `_helpers.tpl:26-40` instead of "securityContext blocks" |
| V-13 | ALB HTTP only (E-35, confidence M) | **Live** listener list: `80 HTTP` only; module adds 443 only when `enable_https` (`modules/alb/main.tf:32`) | **Verified** | Raise E-35 to High; cite the live listener |
| V-14 | "ALB open 0.0.0.0/0" and `security-groups/main.tf:15-30` (E-34) | Lines 15-30 are the **control-plane** HTTPS ingress (`control_plane_ingress_cidrs`) and control-plane **egress** `0.0.0.0/0`, not the ALB | **Partially verified** | Cite `alb_ingress_cidrs = ["0.0.0.0/0"]` in `dev/terraform.tfvars` (CI/CD section) and the ALB module ingress rules (`modules/alb/main.tf:29-32`) for ALB exposure |
| V-15 | JWT verification (E-29) | `verify` pins `algorithms=[settings.jwt_algorithm]`, issuer, audience, required claims (`tokens.py:15,52-56`) and leeway | **Verified** | — |
| V-16 | Uniform `require_permission` (E-30) | identity and patient take `*permissions` (all required); provider, facility, scheduling and appointment take a single `permission` | **Partially verified** | Record the signature drift (see F-10 duplication) |
| V-17 | No retry, circuit breaker or idempotency in code (E-38) | Repo-wide search re-run: only the messages at `appointment_service.py:22`, `patient_service.py:71`, `provider_service.py:139` and seed idempotency `seed_service.py:37` | **Verified** (absence) | — |
| V-18 | Timeouts 3 s (E-37) | Code default 3.0 and Helm explicitly sets `"3"` (`values.yaml:52` for scheduling and appointment) | **Verified** (config reaches runtime: live ConfigMap shows 3) | — |
| V-19 | Argo CD automated sync, retry limit 10 (E-44) | `deploy/argocd/apps/values.yaml:21-22` | **Verified** | — |
| V-20 | Datadog SSI enabled (E-46) | `deploy/datadog/values.yaml` `apm.instrumentation.enabled: true`, target `smart-hospital-python` | **Verified** (config) | Live trace presence in Datadog not checked (gap B-2) |
| V-21 | SLO targets 99.5 % and 500 ms are "Terraform module defaults" (E-47; gap K-3) | Module defaults are 99.5/500/99.5, **and** `environments/dev-observability/terraform.tfvars:16-19` sets them explicitly (+ `sla_target = 99.0`) | **Contradicted** (wording) | Say "explicitly configured in dev-observability tfvars". Whether the business agreed them remains Unknown |
| V-22 | 7 Datadog monitors (E-48) | 7 `resource "datadog_monitor"` in `monitors.tf` | **Verified** (config) | Live existence not checked (B-2) |
| V-23 | "415 test functions / 51 files" (E-50) | The function count is correct, but **pytest collects 140 tests in patient-service** versus 53 functions (parametrisation) | **Partially verified** | Report collected test counts per service (`pytest --collect-only -q`), not function counts |
| V-24 | Live: 11 Argo CD apps Synced/Healthy at `dbb032e`, images `5f077cdb0217` (E-23) | Re-checked: 11 Synced/Healthy; root revision `dbb032e8fbb4`; all 6 deployments use `…/<svc>:5f077cdb0217` | **Verified** (2026-11-03) | — |
| V-25 | R7 gitops is stale vs the monorepo only in `db-bootstrap/dev-values.yaml` (inventory §A) | The local checkout of R7 was on branch `devin/1793640500-cicd-rewire`, so a working-tree diff also showed the unmerged Argo CD/README edits. Comparing R7 `main` directly: new ARN present in R7 main = **0** (0 = stale) | **Partially verified** | Re-run the diff as `main` vs monorepo `main` for all of `deploy/` and record the result |
| V-26 | Client contract tests (E-40, E-41) | Both clients are tested with `httpx.MockTransport` handlers and in-memory fakes; there is no test against the real provider service | **Verified** (MOCK only) | Contract claims rest on mocks; mark caller/callee compatibility as "consumer-side only" |
| V-27 | Dead code | Client `close()` methods are used (`main.py:39`, `:42`). `EXTRA_HOURS` is used in `slot_service.py:84` | **No dead code found** in the inspected paths | A full dead-code scan was not run (vulture/coverage) |
| V-28 | Environment-specific code paths | No `Environment.PRODUCTION` branches in `main.py` or `config.py` beyond the enum; behaviour differences are flag-driven (V-04) | **Verified** | — |

## 2. Cross-document contradictions found

| # | Documents | Contradiction | Resolution |
|---|---|---|---|
| X-1 | Discovery §6 vs trade-off C-7 / ADR E-7 | SES "provisioned" vs "not created without a domain" | Discovery is wrong (V-08) |
| X-2 | ARB review C-1 vs current live state | "SQS/RDS/EKS not deployed" (true on 2026-11-02) vs deployed (2026-11-03) | Both correct **at their date**; every live claim needs a timestamp |
| X-3 | Dependency map G-2 vs discovery §10 | Fail-closed "Strongly indicated" vs "Confirmed" | Verified (V-05) |
| X-4 | Evidence pack K-3 vs dev-observability tfvars | "May be module defaults" vs explicitly configured | V-21 |

## 3. Blocking evidence gaps (must close before options are generated)

| # | Gap | Why blocking | Evidence needed |
|---|---|---|---|
| B-1 | No business-agreed targets (latency, volume, availability, loss/duplicate tolerance, RPO/RTO) | Options can't be scored against unknown targets | Decision-brief §5 answers from the product owner / SRE |
| B-2 | Live Datadog state (SLOs, monitors, traces) not inspected | Observability claims are config-only | Read-only Datadog API query of SLO and monitor IDs plus a trace sample per service |
| B-3 | Jira (KAN/UXBA) and qTest 32916 not inspected | Requirements may already exist | Search for notification/booking requirements and linked test cases |
| B-4 | Caller/callee compatibility is proven only by consumer-side mocks | Contract drift between repos is undetectable | A provider-side contract or integration test (or an exported OpenAPI diff) for D-01 and D-02 |
| B-5 | R7 vs monorepo drift not measured on `main` (V-25) | The deployment source of truth is ambiguous | Branch-to-branch diff, plus a decision on which repo is authoritative |
| B-6 | Patient consent and contact-preference ownership unknown | It drives privacy design | patient-service owner / privacy input |

**Non-blocking corrections:** V-10, V-12, V-13, V-14, V-16, V-21, V-23, X-1. Apply them to the original documents in a
separate change; this report doesn't edit them.
