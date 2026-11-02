# Architecture Pack Quality Review (independent ARB review)

| Field | Value |
|---|---|
| Documents reviewed | `cross-repository-discovery.md` (PR #1, branch `devin/1793638443-cross-repo-discovery`), `dependency-mapping.md` (`devin/1793641500-dependency-mapping`), `architecture-options-tradeoff.md` (`devin/1793642500-notification-options`), `ADR-001-notification-architecture.md` (`devin/1793643500-adr-001-notification`) |
| Reviewer role | Independent Architecture Review Board reviewer |
| Method | Claims re-checked against the 9 `UST-HealthCare-Project` repositories (`main`). A **read-only** AWS check in account `165742853391`, `us-east-1` (`sts get-caller-identity`, `sqs list-queues`, `rds describe-db-instances`, `eks list-clusters`, `ecr describe-repositories`, `secretsmanager list-secrets`, `iam get-role`, `events list-event-buses`, `sesv2 list-email-identities`) on 2026-11-02 |
| Outcome | **ADR-001 is not ready for approval.** 3 Critical, 8 High, 12 Medium and 6 Low findings. The ADR was **not edited**, and its status stays **Proposed** |

Severity: **Critical** blocks approval because the decision may be wrong or unsafe. **High** must be fixed before
approval. **Medium** should be fixed before implementation. **Low** is editorial or a clarification.

---

## 1. Review-gate status (recommended execution sequence)

| Order | Document | Gate | Status | Blocking findings |
|---|---|---|---|---|
| 1 | Discovery | Repository coverage and evidence quality | **Not passed** | C-1, H-7, M-1 |
| 2 | Dependency mapping | Important dependencies and impact scope | **Conditionally passed** | M-2, L-3 |
| 3 | Options trade-off | Requirements, criteria and assumptions agreed | **Not passed** | H-6, M-7, M-8 |
| 4 | ADR draft | Proposal follows from evidence and trade-off | **Not passed** | C-1, C-2, C-3, H-1 to H-5, H-8 |
| 5 | Quality review | Findings resolved, human approval | **Open** | All of the above; §5 questions |

---

## 2. Findings

### Critical

**C-1 — "Provisioned" infrastructure is defined in Terraform but not deployed.**
- *Evidence:* The trade-off (C-5, C-6, §8: "reuses the provisioned SQS and DLQ"), the ADR (E-5, §7 rationale,
  §8 "existing" SQS/DLQ) and the dependency map (D-12, D-13, §7 SPOFs) describe SQS, RDS, EKS and IAM as existing.
  The read-only AWS check on 2026-11-02 found, in `165742853391`/`us-east-1`: **no SQS queues, 0 RDS instances,
  0 EKS clusters, 0 `smart-hospital*` ECR repositories, 0 `smart-hospital*` secrets, no
  `smart-hospital-dev-github-actions` role, only the `default` event bus, and no SES identities.** The resources exist
  only as code in `smart-hospital-infra:infra/terraform/environments/dev/*`.
- *Impact:* The ADR's main rationale for Option C ("reuse provisioned infrastructure", driver R6) and the migration
  effort scores aren't supported by live state. Cutover steps that assume a running dev stack (CI/CD rewiring, Argo CD)
  can't be executed as written.
- *Correction:*
  - Relabel these as "defined in Terraform; not deployed as of <date>" in all four documents.
  - Rescore R6 and migration effort for all options.
  - Add a "live-state verification" step to the ADR approval criteria.

**C-2 — The deduplication key isn't in the event contract.**
- *Evidence:* ADR §7 and §10 dedupe on the "outbox event `id`". The payload contract (ADR E-3;
  `appointment-service:app/services/appointment_service.py:209-230`) doesn't contain the outbox `id`
  (`OutboxEvent.id`, `app/models/outbox_event.py:25`). It has no `schema_version` either. EventBridge assigns its own
  event id per `PutEvents` call, so a relay retry after a lost acknowledgement creates a **different** bus event id.
- *Impact:* R3 (duplicate safety) isn't achievable as designed. Patients can receive duplicates on relay retries.
- *Correction:*
  - Make `event_id` (= `outbox_events.id`) and `schema_version` mandatory payload fields in the ADR, as a contract
    change to appointment-service.
  - Dedupe on `event_id` only.
  - Add a contract test.

**C-3 — The consumer's service identity is unresolved, on top of a shared signing key.**
- *Evidence:*
  - ADR §9 and Q5: notification-service has no way to call patient-service. Today only the user's token is forwarded
    (`scheduling-service:app/clients/facility_client.py:66-70`; F-08).
  - All services verify with one HS256 key (`smart-hospital-infra:modules/secrets/main.tf:34-49`; F-07), so any
    holder of that key can mint tokens with any role.
- *Impact:*
  - R4 (contact lookup) can't be implemented.
  - Any shortcut, such as giving notification-service the shared key to mint its own token, would widen the blast
    radius to every service.
  - The decision depends on a security design that hasn't been made.
- *Correction:*
  - Decide the service-identity mechanism before approval, in a prerequisite ADR-002 or in ADR-001 itself. Options
    include client-credentials tokens issued by identity for a dedicated role, and asymmetric per-issuer keys with a
    JWKS endpoint.
  - Make security sign-off of that decision an explicit precondition.

### High

**H-1 — There is a crash window between recording a request and sending it.**
- *Evidence:* ADR §10: "delivery adapter called only on first insert."
- *Impact:* If the process crashes after the insert commits and before the send, the request never goes out (silent
  loss). If it sends first and then records, a crash causes duplicates.
- *Correction:*
  - Specify a status state machine (`PENDING → SENDING → SENT | FAILED`) with retry driven by status and a
    max-attempts limit.
  - Pass an idempotency key to the provider where the provider supports one (validate per provider).

**H-2 — First activation would flood notifications for old appointments.**
- *Evidence:* Every booking since go-live has `published_at IS NULL` (discovery §6; E-4). ADR §15 S1/S4 enable a relay
  that publishes all unpublished rows.
- *Impact:* Patients get notifications for past, cancelled or rescheduled appointments, along with a load spike.
- *Correction:*
  - Define a cut-over watermark: mark rows before activation as `published_at = skipped`, or filter on `occurred_at`.
  - Have the consumer skip events whose `starts_at` is in the past.

**H-3 — The rollback window is bounded by SQS retention.**
- *Evidence:* Main queue retention defaults to `345600` s (4 days) (`smart-hospital-infra:modules/sqs/variables.tf:12-15`).
  ADR §15 S2 rollback is "scale to 0" while the relay keeps marking rows published.
- *Impact:* Messages older than 4 days expire without reaching the DLQ, and their outbox rows already say published.
  That is silent loss.
- *Correction:*
  - The rollback runbook must **pause the relay first** (flag off), then stop consumers.
  - Set queue retention to cover the agreed recovery time, or re-drive from the outbox.

**H-4 — The EventBridge target needs a merged SQS queue policy.**
- *Evidence:* The queue's only policy is `aws_sqs_queue_policy.tls_only` (`modules/sqs/main.tf:29-55`). ADR §12 adds
  "a queue policy allowing `events.amazonaws.com`".
- *Impact:* A second `aws_sqs_queue_policy` replaces the first. Either the TLS-deny statement is lost or EventBridge
  can't deliver.
- *Correction:* Extend the module's single policy document with an `events.amazonaws.com` `sqs:SendMessage`
  statement, conditioned on the rule ARN. Add a `terraform test` assertion.

**H-5 — The privacy design for contact data is unspecified.**
- *Evidence:* ADR §9 and §10 don't say whether notification-service stores contact details (email/phone) in
  `notification_requests`, for how long, who can read them, or how logs are redacted.
- *Impact:* Patient contact data could be copied into another service's database and logs, against the minimum-data
  rules (`AGENTS.md:41-45`; `Smart_Hospital_Repository_Rules.md:34`).
- *Correction:*
  - Resolve contact details at send time and never persist them, or persist only hashed or masked values.
  - Define retention.
  - Add log-redaction tests following `appointment-service:tests/test_logging.py`.

**H-6 — The option comparison isn't like-for-like.**
- *Evidence:*
  - In the trade-off §7, Option A scores 2/2 on R1/R2 assuming **no** relay, while B and C are scored **with** the
    outbox relay. The ADR §5 itself states that all options need the relay.
  - The variant OC′ (SQS direct) is recommended for the POC but never scored.
  - "Do nothing" appears only in the ADR.
- *Impact:* The ranking (A 320 / B 355 / C 415) partly reflects the framing, not the transports.
- *Correction:* Rescore A with a relay, add OC′ and O0 to the matrix, and score transports only on the criteria they
  actually differ on.

**H-7 — Evidence comes from one Git snapshot, without live or adjacent sources.**
- *Evidence:*
  - The discovery scope is a split from monorepo commit `63bf585`. The live system deploys from the monorepo (F-01),
    and the CI/CD rewiring branches aren't merged.
  - Discovery §13 lists no live cluster, Argo CD, RDS or Datadog inspection; no frontend repo; Jira/qTest requirements
    not inspected.
- *Impact:* The pack may describe a state that differs from both the running system and newer monorepo commits.
- *Correction:* Record the source SHA in every document, re-verify against the monorepo `main` and live state before
  approval, and add the missing sources or mark them out of scope.

**H-8 — Disaster recovery is undefined.**
- *Evidence:*
  - No RPO/RTO in any document.
  - Bookings and the outbox share one single-AZ RDS instance (`rds_multi_az = false`,
    `rds_backup_retention_period = 7`, `smart-hospital-infra:.../dev/terraform.tfvars:103-104`).
  - The dedupe store would live in a separate database restored to a different point in time.
- *Impact:* A point-in-time restore can re-publish events (duplicates) or lose `published_at` state. The pack can't
  show recovery behaviour.
- *Correction:* Define RPO/RTO, the restore procedure for outbox and notification databases, dedupe behaviour across
  restores, and whether a cross-region design is required.

### Medium

| ID | Finding | Evidence | Impact | Correction |
|---|---|---|---|---|
| M-1 | **SES contradiction between documents** | The discovery §6 row says the "SES domain identity … provisioned". The trade-off C-7 and ADR E-7 say SES is created only with a custom domain, and dev has `hosted_zone_name = null` (`dev/main.tf:134-136`; `dev/terraform.tfvars:111`). The live check found no SES identity | Readers get opposite answers | Correct the discovery wording |
| M-2 | **Scheduling fail-closed is understated** | The dependency map D-02 and G-2 say "Strongly indicated … no test", while the discovery §10 says Confirmed and tests exist (`appointment-service:tests/test_scheduling_advice.py` imports `ServiceUnavailableError`) | Inconsistent confidence | Cite the test; align to Confirmed (after running it) |
| M-3 | **Ordering rule can drop the create event** | ADR §10: ignore events whose `version` ≤ the stored one | If a cancel (v2) arrives first, the create (v1) is ignored, so the patient gets only "cancelled" | Define per-type semantics (suppress the cancel if no create was sent; or reconcile by reading the appointment state) |
| M-4 | **Relay poison message / head-of-line blocking** | The relay polls by `occurred_at` (ADR §8, §11). There is no attempt count or parking | One permanently rejected row can block or loop forever | Add attempt counters, parking state, an alert and batch isolation |
| M-5 | **Queue settings not sized to consumer work** | `visibility_timeout_seconds = 60`, `max_receive_count = 5` (`modules/sqs/variables.tf:6-21`) | Patient lookup + provider latency could exceed the visibility timeout, causing duplicate receives and early DLQ | Size after POC measurement; document in the ADR |
| M-6 | **No observability for the new flow** | The existing monitors cover SLO burn, latency, replicas, restarts, nodes and error logs only (`modules/datadog-observability/monitors.tf:15-95`). Nothing emits outbox lag | ADR §12 monitors have no data source | Specify the emitters (custom metric or SQL check), queue-depth and DLQ metrics, and make them POC exit criteria |
| M-7 | **Unsupported performance, scalability, operations and cost assertions** | See §3 | Scores rest on unvalidated statements | Mark as hypotheses; give cost weight 0 until estimated |
| M-8 | **Weights set by the proposer, with no sensitivity analysis** | Trade-off §7 | Bias risk. Example: swapping the operations (15) and portability (5) weights gives A 320 / B 385 / C 395, so the margin drops from 60 to 10 | ARB agrees the weights independently; include a sensitivity table |
| M-9 | **The event schema is unversioned** | No `schema_version` (E-3); ADR Q8 open | Future consumers (encounter, analytics) couple to an implicit contract | Decide versioning with C-2 |
| M-10 | **Inputs aren't frozen; links are broken** | All four documents are on separate unmerged branches; the relative links fail until merge | The ADR may be approved against text that later changes | Merge in order (discovery → dependency → trade-off → ADR → review) or pin commit SHAs in the ADR header |
| M-11 | **The approval body isn't named** | ADR §19 lists roles only | No accountable approver; no decision date or expiry | Name the accountable architect and governance body; add a review-by date |
| M-12 | **Event scope is wider than the problem** | The problem is "after successful bookings"; the ADR covers create, reschedule and cancel (§8 rule) | Scope creep without product confirmation (Q3) | Limit the decision to `AppointmentCreated`, or get Q3 answered |

### Low

| ID | Finding | Evidence | Correction |
|---|---|---|---|
| L-1 | Token lifetime phrasing differs: the discovery diagram says "exp≤60m"; the dependency map says "15 min default" | `identity-service:app/core/config.py:48` (default 15, max 60); `smart-hospital-gitops:deploy/helm/identity-service/values.yaml:50` (`"15"`) | State "15 min configured, 60 max" in both |
| L-2 | "Copied six times and drifted" overstates the drift | `app/core/tokens.py` is identical in 4 of 6 services (md5 `d2b91957`); patient and identity differ | Say which files differ |
| L-3 | Direction notation differs: the dependency map Mermaid shows issuer → verifiers, while matrix D-04 lists verifiers → identity | `dependency-mapping.md` §2 vs §3 | Add a direction convention to the legend |
| L-4 | Mixed path conventions (some monorepo `services/<svc>/…` paths in the discovery, split-repo paths elsewhere) | Discovery header "Path convention" | Normalise to `repo:path` |
| L-5 | The relay design uses `FOR UPDATE SKIP LOCKED`, but service tests default to SQLite | ADR §11; `appointment-service` CI runs SQLite then PostgreSQL | Require PostgreSQL tests for the relay |
| L-6 | EventBridge → SQS behaviour with SSE-SQS, and the target-DLQ policy, are not validated | ADR §9, §12 | Keep under §17 provider validation; add it to POC P2 |

---

## 3. Unsupported performance, availability and cost claims

| Location | Claim | Status | Required action |
|---|---|---|---|
| Trade-off §4 "Performance" (B) | "Low-latency broker" | Unsupported | Measure in the POC or remove |
| Trade-off §4 "Scalability" (C) | "Managed; scale consumers on SQS depth" | Unsupported as a capacity claim (quotas not checked) | Provider validation (§9) |
| Trade-off §4 "Operational support" (C) | "Low: managed service" | Qualitative and unsupported | Turn it into a list of operational tasks per option |
| Trade-off §4 "Disaster recovery" (C) | "Regional service" | Correct description, but gives no availability evidence | Cite provider documentation at decision time |
| Trade-off §7 "Cost" row (5/3/4) | Relative cost ranking | Unsupported (no estimate exists) | Weight 0 until a Pricing Calculator estimate exists |
| Trade-off §8, ADR §7 | "Reuses the provisioned SQS and DLQ" | **Contradicted by live state** (C-1) | Relabel and rescore |
| Discovery §6 | "SES domain identity … provisioned" | Contradicted by config and live state (M-1) | Correct |
| ADR §11 | Booking-path latency "close to zero" | Correctly labelled [Hypothesis] | Keep; measure in P3 |
| Dependency map §7 | ALB/WAF "managed service" mitigation | Correct description; no SLA asserted | None |

No benchmark, price or SLA figure appears in any document. The findings above concern **comparative** or
**qualitative** statements that are scored as if validated.

## 4. Areas reviewed with no finding

- API contracts for D-01 and D-02 match the client code (`facility_client.py:35-87`, `scheduling_client.py:67-100`).
- Database ownership (one DB and role per service; no cross-service FKs) matches the migrations and bootstrap SQL.
- Producer/consumer direction for the outbox is correct.
- The ADR keeps **Status: Proposed** and makes acceptance conditional on approval criteria.
- The Care Points routing conflict (`/api/v1/patients*`) is correctly flagged as Strongly indicated.

## 5. Questions the architect must resolve before approval

1. **Live state:** will dev be rebuilt before the POC, and which environment is the reference for evidence (C-1, H-7)?
2. **Service identity:** which mechanism lets notification-service call patient-service, and does it remove the
   shared-key blast radius (C-3)?
3. **Event contract:** confirm `event_id` and `schema_version` as mandatory payload fields, and who owns the schema (C-2, M-9).
4. **Delivery state machine:** what are the accepted at-least-once/duplicate trade-offs at the provider boundary (H-1)?
5. **Backlog policy:** what happens to existing unpublished outbox rows at first activation (H-2)?
6. **Rollback:** confirm the order (pause relay → stop consumer) and the queue retention needed (H-3).
7. **Privacy:** may notification-service persist contact data? Retention and access (H-5)?
8. **Ordering semantics** for cancel/reschedule arriving before create (M-3); and is scope `AppointmentCreated` only (M-12, ADR Q3)?
9. **Decision criteria:** who sets the weights? Rescore with A-with-relay, OC′ and O0 included (H-6, M-8).
10. **DR:** RPO/RTO for bookings and notifications; is single-AZ acceptable beyond dev (H-8)?
11. **Observability:** which metrics and monitors are acceptance criteria for P2/P3 (M-6)?
12. **Governance:** who is the accountable approver, and what is the review-by date (M-11)? Is a managed event bus
    approved under `AGENTS.md:35,60`?

## 6. Recommended corrections (no document was rewritten in this review)

| Document | Correction | Findings |
|---|---|---|
| Discovery | Fix the SES statement; add the source SHA; separate "defined in Terraform" from "deployed" | C-1, M-1, H-7, L-1, L-2, L-4 |
| Dependency mapping | Cite the scheduling-advice tests; add a direction legend; label deployment state | M-2, L-3, C-1 |
| Options trade-off | Rescore A with a relay; add OC′ and O0; cost weight 0; sensitivity table; relabel qualitative claims | H-6, M-7, M-8, C-1 |
| ADR-001 | Add `event_id` and `schema_version`; resolve service identity; delivery state machine; backlog watermark; rollback order; merged SQS policy; privacy, DR and observability sections; named approver | C-2, C-3, H-1 to H-5, H-8, M-3 to M-6, M-9 to M-12 |

**ADR-001 status remains Proposed.** This review does not accept, reject or modify it.
