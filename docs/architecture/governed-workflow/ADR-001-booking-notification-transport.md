# ADR-001: Booking notification transport

## 1. Identity

| Field | Value |
|---|---|
| ID | ADR-001 (governed workflow) |
| Title | Booking notification transport between appointment-service and the notification capability |
| Date | 2026-11-03 |
| **Status** | **Proposed, with blockers.** The final selection between the two shortlisted options is **Deferred** until blockers BL-1 to BL-5 close (§8). **Not Accepted** |
| Decision owner role | Accountable / principal architect (person: **Open**, brief §7) |
| Inputs | `decision-brief.md`, `repository-inventory.md`, `current-state-architecture.md`, `evidence-register.md` (E-nn), `evidence-validation-report.md` (V-nn, B-n), `option-set.md` (O0–O5, P-1–P-6), `four-axis-tradeoff.md` |

**Labels:**
- **[Fact]** is verified evidence.
- **[Assumption]** is accepted for planning, not verified.
- **[Unknown]** is missing evidence.
- **[Proposal]** is this ADR's position.

## 2. Context and current-state evidence

- [Fact] appointment-service writes `AppointmentCreated/Rescheduled/Cancelled` rows to `outbox_events` in the booking
  transaction, and `published_at` is only ever set to `None` (`appointment-service:app/services/appointment_service.py:188-230`; V-06).
- [Fact] There is no notification-service code or repository (inventory C8).
- [Fact] Contact data lives only in patient-service (`patient-service:app/schemas/patient.py:95-105`; E-14).
- [Fact] Service clients forward the user's token; there is no service identity (`scheduling-service:app/main.py:30`;
  `appointment-service:app/main.py:32-35,62`; V-03, V-04).
- [Fact] No retry, circuit-breaker or idempotency code exists (V-17). Client timeouts are 3 s and reach the runtime
  (V-18).
- [Fact] Live dev (2026-11-03): 11 Argo CD apps Synced/Healthy at monorepo `dbb032e8fbb4`, an HTTP-only ALB, and 2
  SQS queues that exist but are unused (V-07, V-13, V-24).

## 3. Business problem and decision question

- **Problem:** patients aren't notified after booking. Booking must stay available when notification isn't (brief
  §1.1–1.2).
- **Question:** which transport pattern should carry committed booking events from appointment-service to the
  notification capability?

## 4. Scope, requirements, constraints and drivers

- **Scope:** the transport, delivery guarantees, duplicates and ordering, contact-data access, operations, migration.
  Out of scope: product choice, real channels, clinical content (brief §3).
- **Requirements:** booking isolation; no silent loss; no duplicate messages; per-appointment ordering (to confirm).
  **All numeric targets are Open** (brief §5; B-1).
- **Constraints:**
  - each service owns its data (`AGENTS.md:30-31`);
  - transport-independent, mock adapter first, a broker only with approval (`AGENTS.md:35,60`);
  - idempotency (`Smart_Hospital_Repository_Rules.md:49`);
  - IDs-only, non-clinical content (`AGENTS.md:45`).
- **Drivers:** D1–D12 (brief §6). **Weights not agreed.**

## 5. Evidence summary

| Topic | Reference | Status |
|---|---|---|
| Outbox write in the booking tx | `appointment-service:app/services/appointment_service.py:188-207` (`_save`) | Verified (V-06) |
| Payload lacks `event_id` and `schema_version` | `appointment_service.py:209-230` (`_event`) | Verified |
| Optional scheduling advice, live true | `appointment-service:app/main.py:32-62`; `deploy/helm/appointment-service/dev-values.yaml:19-20` | Verified (V-04) |
| Fail-closed advice | `app/clients/scheduling_client.py:82-99`; `tests/test_scheduling_advice.py` | Verified (V-05) |
| SQS defaults 60 s / 4 d / 5 | `smart-hospital-infra:infra/terraform/modules/sqs/variables.tf` | Verified (V-07) |
| RDS single-AZ, 7-day backups | `environments/dev/terraform.tfvars:99-106` | Verified (V-09) |
| No HPA/PDB; dev replicas 1 | `deploy/helm/*-service/dev-values.yaml:8` | Verified (V-11) |
| Datadog monitors (7), none for queues/outbox | `modules/datadog-observability/monitors.tf` | Verified config (V-22); live not checked (B-2) |
| Client contracts tested with mocks only | `FakeFacilityClient`, `FakeSchedulingClient`, `httpx.MockTransport` | Verified (V-26) |

## 6. Options considered

| ID | Pattern | Architecture (summary; full detail in `option-set.md`) |
|---|---|---|
| O0 | Do nothing | Outbox rows written; nothing delivered |
| O1 | Retain and improve: outbox + in-service relay + synchronous push | The relay in appointment polls unpublished rows and POSTs to notification with idempotency key `event_id` |
| O2 | Incremental: outbox + relay + durable point-to-point queue | The relay publishes to a queue with DLQ; the notification consumer dedupes and acks |
| O3 | Different: consumer-pull change feed | appointment exposes a read-only, ordered event feed; notification polls with its own cursor |
| O4 | Different: publish/subscribe backbone | The relay publishes to a topic; each consumer has a subscription + DLQ |
| O5 | Hybrid by workload class | Transactional events via O2/O3; scheduled reminders via a batch read |

## 7. Four-axis comparison (from `four-axis-tradeoff.md`)

| Option | Latency | Cost | Consistency | Operability |
|---|---|---|---|---|
| O0 | n/a (no delivery) | 5 | n/a | 5 |
| O1 | Unknown end-to-end (booking path 5) | 4 | 4 | 4 |
| O2 | Unknown | Unknown | 3 | 3 |
| O3 | Unknown (poll-interval floor) | 4 | 4 | 4 |
| O4 | Unknown | Unknown | 3 | 2 |
| O5 | Unknown | Unknown | 3 | 2 |

## 8. Proposed decision and rationale

**[Proposal]**

1. Adopt the **transport-independent foundation P-1 to P-6** now, for any option:
   - P-1: `event_id` + `schema_version` in the payload;
   - P-2: a notification store with dedupe;
   - P-3: a service identity;
   - P-4: a backlog watermark;
   - P-5: targets;
   - P-6: lag and DLQ monitoring.
2. **Shortlist O1 and O3. Defer the choice between them** to a measured POC.

**Rationale (evidence-linked):**
- O1 and O3 are the only options with no Unknown on cost, consistency and operability. They need **no broker
  approval** (`AGENTS.md:35,60`), and they reuse the outbox (V-06) and the existing deployment path (V-24).
- Choosing between them depends on the end-to-end latency target and measurements, which don't exist (U-1, B-1).
  O3's poll-interval floor versus O1's point-to-point coupling can't be settled without them.

**Blockers before Accepted:**

| ID | Blocker | Source |
|---|---|---|
| BL-1 | Agreed targets and driver weights | B-1 |
| BL-2 | Service-identity design approved by security | P-3; brief Q7 |
| BL-3 | POC measurements for O1 and O3 (p50/p95/p99, duplicates and losses after fault injection) | four-axis §7 |
| BL-4 | Named SLO and on-call owners | U-4 |
| BL-5 | Jira/qTest and live Datadog inspected for existing requirements and telemetry | B-2, B-3 |

## 9. Rejected options and reasons

| Option | Rejected against requirements | Rejected against the four axes |
|---|---|---|
| O0 | Fails the business outcome: no patient is notified (brief §1.1) | Its axis ratings are high only because it does nothing |
| O2 | **Not rejected outright; deferred.** Needs broker approval (`AGENTS.md:35,60`) that hasn't been given | Cost Unknown (pricing); consistency 3 (no ordering, redelivery duplicates, retention-bounded rollback); operability 3 (new DLQ/redrive surface, no monitors today) |
| O4 | Fan-out isn't a confirmed requirement (brief Q2); needs broker approval | Cost Unknown; consistency 3; operability 2 (most hops, schema governance, per-subscriber DLQs) |
| O5 | Distinct workload classes aren't confirmed (brief Q2/Q3; reminders out of scope) | Cost Unknown; consistency 3 (two dedupe domains); operability 2 (two mechanisms) |

## 10. Conditions under which a rejected option becomes viable

| Option | Becomes viable if… |
|---|---|
| O2 | A broker is approved, pricing and quotas at the agreed volume are acceptable, **and** the POC shows O1/O3 can't absorb notification outages within targets |
| O4 | Two or more confirmed consumers of booking events (e.g. encounter Phase 7, analytics) and schema governance has an owner |
| O5 | The product owner confirms scheduled reminders with different delay and volume tolerances |
| O0 | Never as an end state; only as the rollback position |

## 11. Consequences, accepted trade-offs and new dependencies

- **Consequences:** the event payload becomes a versioned contract; a new notification service and store; a new
  notification → patient sync dependency; eventual (not immediate) notification.
- **Accepted trade-offs:** at-least-once delivery with consumer dedupe instead of exactly-once; no broker for now,
  in exchange for hand-built retry (O1) or polling (O3).
- **New dependencies:** a service identity issued or validated by identity; a patient contact read API with a narrow
  contract; Datadog monitors for lag and DLQ.

## 12. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Crash between storing a request and sending it (loss or duplicate) | Status state machine `PENDING → SENT/FAILED` with retries (ARB H-1) |
| Flood of old events on first activation | Watermark (P-4) and skipping past `starts_at` |
| Ordering errors (a cancel before its create) | Version check with per-type rules (ARB M-3); O3 sequence ordering |
| Shared HS256 key blast radius grows with service identity | Security design under BL-2; prefer asymmetric per-issuer keys (F-07) |
| Extra DB load on single-AZ RDS (O3) | Measure in the POC; read index on the sequence |
| Coupling and retry storms (O1) | Backoff with cap; circuit-breaking on notification errors |

## 13. Security, privacy, resilience and operations

- **Security:** service identity per BL-2; least-privilege permissions for the feed or POST endpoints; no new
  long-lived secrets in Git (gitleaks; E-33).
- **Privacy:** events stay IDs-only (E-10). Contact details are read at send time and not persisted beyond what the
  privacy reviewer approves (brief Q6). Logs exclude contact data (E-45).
- **Resilience:**
  - booking is isolated, because the event is already committed in the booking transaction;
  - replay is from the outbox, or from the cursor in O3;
  - recovery is bounded by RDS backups (7 days, single-AZ; RPO/RTO **Open**).
- **Operations:**
  - add monitors for outbox lag, the oldest unpublished event, notification failures and DLQ depth;
  - add runbooks for redrive and replay;
  - no HPA/PDB exists (V-11), so the replica policy must be decided.

## 14. Migration, rollout, rollback and validation

| Stage | Change | Default | Rollback |
|---|---|---|---|
| S0 | Approve the blockers | — | — |
| S1 | P-1 payload fields (additive) + watermark | — | Revert (additive) |
| S2 | notification-service with mock adapter + store + dedupe | Flag OFF | Scale to 0 |
| S3 | POC of O1 and O3 in dev, behind flags | OFF | Flag off; rows remain |
| S4 | Select O1 or O3 by measured results; update this ADR | — | — |
| S5 | Enable in dev → higher environments | OFF → ON | Pause relay/poller first, then consumers |

**Validation:**
- booking returns 201 with notification down;
- exactly one notification per event under injected duplicates;
- stale events are ignored;
- end-to-end latency is measured against the agreed targets;
- no lost events after a relay/consumer crash (outbox count = request count).

## 15. Assumptions, unknowns and open questions

- **Assumptions:**
  - A-1: a delay of minutes is acceptable;
  - A-2: notification runs on the existing nodes;
  - A-3: demo-scale volume.
- **Unknowns:** U-1 latency target and measurements; U-2 managed pricing; U-3 fan-out need; U-4 SLO and on-call
  owners; U-5 broker approval; U-6 RPO/RTO.
- **Open questions:** brief Q1–Q12, especially Q2 (which events), Q3 (channels), Q6 (consent), Q7 (identity) and
  Q9 (lock-in).

## 16. Required reviewers and approval checklist

| Reviewer role | Name |
|---|---|
| Decision owner (accountable architect) | **Open** |
| Product owner | **Open** |
| Security & privacy | **Open** |
| appointment-service owner | **Open** |
| patient-service owner | **Open** |
| Platform / SRE | **Open** |
| ARB / governance body | **Open** |

**Checklist:**
- [ ] BL-1 to BL-5 closed, with evidence attached
- [ ] O1 vs O3 selected from POC measurements and §8 updated
- [ ] Security sign-off on service identity and contact-data handling
- [ ] Monitors and runbooks defined, with named owners
- [ ] Rollback rehearsed in dev
- [ ] The status change to Accepted is recorded by the decision owner

## 17. Reassessment triggers and review date

- **Triggers:**
  - a second consumer of booking events is confirmed (re-evaluate O4);
  - a broker is approved (re-evaluate O2);
  - measured latency or loss misses its targets;
  - volume grows beyond the POC assumptions;
  - the platform changes (RDS Multi-AZ, a different cluster);
  - the security key strategy changes.
- **Review date:** **Open.** To be set by the decision owner when the blockers are assigned. Suggested: at the end of
  the POC (S3/S4).

**Status: Proposed, with blockers. Selection Deferred between O1 and O3.**
