# Human Review Pack: ADR-001 Booking Notification Transport

| Field | Value |
|---|---|
| Prepared by | Architecture Review Coordinator (Devin). **Coordinator only: no outcome chosen, no approval recorded** |
| Decision owner | Architecture Decision Owner. **Name: Open, to be supplied by the governance body** |
| ADR under review | `ADR-001-booking-notification-transport.md`. **Status: Proposed, with blockers** (unchanged by this pack) |
| Supporting artefacts | `decision-brief.md`, `repository-inventory.md`, `current-state-architecture.md`, `evidence-register.md`, `discovery-gaps.md`, `evidence-validation-report.md`, `option-set.md`, `four-axis-tradeoff.md` |
| Date prepared | 2026-11-03 |

## 1. Decision question

Which transport pattern should carry committed booking events from appointment-service to a notification capability,
so that patients are notified after booking while **booking remains available when notification is not**?
(ADR §3; brief §1)

## 2. Business impact

- **Today** patients receive no booking notification of any kind (brief §1.1; V-06).
- **If unresolved:** missed or duplicate appointments can't be reduced through notifications, and the planned
  notification service (Phase 8, `AGENTS.md`) has no agreed integration.
- **Risk of a wrong choice:**
  - lost notifications (patient not informed);
  - duplicate notifications (confusion);
  - out-of-order messages (a cancellation arriving before its confirmation);
  - booking availability coupled to notification.

## 3. Verified facts (evidence links)

| # | Fact | Evidence |
|---|---|---|
| F1 | Booking events are written to `outbox_events` in the same transaction; nothing publishes them | `appointment-service:app/services/appointment_service.py:188-230`; V-06 |
| F2 | Event payload carries IDs only, with no `event_id` or `schema_version` | `appointment_service.py:209-230` (`_event`) |
| F3 | No notification-service code or repository exists | `repository-inventory.md` C8 |
| F4 | Contact data is owned only by patient-service | `patient-service:app/schemas/patient.py:95-105`; E-14 |
| F5 | No service identity; clients forward the user's token | `scheduling-service:app/main.py:30`; `appointment-service:app/main.py:32-62`; V-03, V-04 |
| F6 | No retry, circuit-breaker or idempotency code; 3 s timeouts reach the runtime | V-17, V-18 |
| F7 | Unused SQS queue + DLQ exist in dev (defaults: 60 s visibility, 4-day retention, 5 receives) | `smart-hospital-infra:infra/terraform/modules/sqs/variables.tf`; V-07 |
| F8 | RDS single-AZ, 7-day backups; no HPA/PDB | `environments/dev/terraform.tfvars:99-106`; V-09, V-11 |
| F9 | Datadog: 7 monitors configured, none for outbox, queue or DLQ | `modules/datadog-observability/monitors.tf`; V-22 |
| F10 | Live dev: 11 Argo CD apps Synced/Healthy at monorepo `dbb032e8fbb4`; HTTP-only ALB | V-13, V-24 (observed 2026-11-03) |

## 4. Assumptions and unknowns

| ID | Type | Statement | Owner role |
|---|---|---|---|
| A-1 | Assumption | A delay of minutes is acceptable for confirmations | Product/Business Owner |
| A-2 | Assumption | Notification runs on the existing cluster capacity | Operations/SRE |
| A-3 | Assumption | Demo-scale volume | Product/Business Owner |
| U-1 | Unknown | End-to-end latency target and measurements | Product + SRE |
| U-2 | Unknown | Managed queue/backbone pricing and quotas | Engineering + Finance |
| U-3 | Unknown | Whether other consumers (fan-out) are required | Architecture Decision Owner |
| U-4 | Unknown | SLO and on-call owners | Operations/SRE |
| U-5 | Unknown | Whether a broker is approved (`AGENTS.md` stop condition) | Governance body |
| U-6 | Unknown | RPO/RTO | Product + SRE |
| U-7 | Unknown | Consent, contact preferences and channels (email/SMS) | Security/Privacy + Product |

## 5. Options and mandatory four-axis comparison

Rating 1 (poor) to 5 (strong). **Unknown is never scored as neutral.** Definitions are in `four-axis-tradeoff.md` §1.

| Option | Pattern | Latency | Cost | Consistency | Operability |
|---|---|---|---|---|---|
| O0 | Do nothing | n/a | 5 | n/a | 5 |
| O1 | Outbox + in-service relay + synchronous push | Unknown (booking path 5) | 4 | 4 | 4 |
| O2 | Outbox + relay + durable point-to-point queue | Unknown | Unknown | 3 | 3 |
| O3 | Consumer-pull change feed (cursor) | Unknown (poll floor) | 4 | 4 | 4 |
| O4 | Publish/subscribe backbone | Unknown | Unknown | 3 | 2 |
| O5 | Hybrid by workload class | Unknown | Unknown | 3 | 2 |

## 6. Recommendation under review (from the ADR, not a decision)

- Adopt the transport-independent foundation P-1 to P-6 (ADR §8).
- **Shortlist O1 and O3.** The selection between them is **Deferred** to a measured POC.
- **Blockers BL-1 to BL-5** must close before the ADR can be considered for Accepted:
  - BL-1: targets and weights;
  - BL-2: security approval of the service identity;
  - BL-3: POC measurements;
  - BL-4: SLO and on-call owners;
  - BL-5: Jira/qTest and live Datadog inspected.

## 7. Rejected and deferred options

| Option | Reason | Viable if |
|---|---|---|
| O0 | Fails the business outcome | Never as an end state (rollback position only) |
| O2 | **Deferred:** broker not approved; cost Unknown; consistency 3; operability 3 | Broker approved + pricing acceptable + POC shows O1/O3 insufficient |
| O4 | Fan-out unconfirmed; broker approval; cost Unknown; operability 2 | Two or more confirmed consumers + a schema governance owner |
| O5 | Workload classes unconfirmed; operability 2 | Reminders confirmed as a distinct class |
| Excluded (option-set §Excluded) | In-transaction call, dual write, shared DB, distributed transactions, change-log tailing, contact data in events | See `option-set.md` |

## 8. Risks, consequences and mitigations

| Risk / consequence | Mitigation (ADR §12) |
|---|---|
| Crash between storing a request and sending it (loss or duplicate) | Status state machine with retries; dedupe on `event_id` |
| Flood of old events on first activation | Watermark; skip past `starts_at` |
| Out-of-order events | Version check; O3 commit-ordered sequence |
| Service identity widens the shared HS256 key blast radius | Security design (BL-2); per-issuer asymmetric keys considered |
| Extra read load on single-AZ RDS (O3) | Measure in the POC; index |
| Retry storms (O1) | Capped backoff; circuit breaking |
| Consequence: the event payload becomes a versioned contract | Schema owner named |
| Consequence: new notification → patient dependency | Narrow contact-read contract with minimum fields |

## 9. POCs and tests required before a decision

| # | Test | Applies to | Pass criterion |
|---|---|---|---|
| T1 | Booking with notification down returns 201 | O1, O3 | 100 % of bookings succeed |
| T2 | Duplicate injection | O1, O3 | Exactly one notification request per `event_id` |
| T3 | Stale and out-of-order events | O1, O3 | Older versions ignored |
| T4 | Relay/consumer crash mid-batch | O1, O3 | Outbox count = request count (no loss) |
| T5 | End-to-end latency p50/p95/p99 (normal + degraded) | O1, O3 (+ O2 if broker approved) | Against the target from BL-1 (**Open**) |
| T6 | First-activation backlog | O1, O3 | Only post-watermark events processed |
| T7 | Rollback rehearsal | Selected option | Flag off → no loss; resume → backlog drains |

## 10. Required approvals

| Role | Approval needed for | Name |
|---|---|---|
| Architecture Decision Owner | The decision outcome and the ADR status change | **Open** |
| Product/Business Owner | Targets, events, channels, reminders scope | **Open** |
| Engineering Owner (appointment, patient, notification) | Contracts, feasibility, effort | **Open** |
| Security/Privacy | Service identity, contact-data handling, consent | **Open** |
| Operations/SRE | SLOs, monitors, runbooks, on-call | **Open** |
| Domain/Clinical Safety | Notification content and timing safety | **Open** (relevant; see §11.6) |
| Governance body (ARB) | Broker approval (only if O2/O4 is revived) | **Open** |

## 11. Review checklists per role

### 11.1 Architecture Decision Owner
- [ ] The decision question (§1) is correctly framed and in scope
- [ ] Verified facts F1–F10 are accepted as the baseline (evidence links checked)
- [ ] The four-axis ratings and the Unknown handling are fair across all options
- [ ] The shortlist (O1, O3) and the deferral of O2 follow from the evidence
- [ ] Rejection reasons and viability conditions for O0/O2/O4/O5 are acceptable
- [ ] Blockers BL-1 to BL-5 have named owners and dates
- [ ] Reassessment triggers and the review date are set (ADR §17)

### 11.2 Product/Business Owner
- [ ] Confirm the decision topic (brief Q1)
- [ ] Provide targets: latency, volume, acceptable loss and duplicates, RPO/RTO (U-1, U-6)
- [ ] Confirm which events notify, and whether reminders are in scope (Q2; affects O5)
- [ ] Confirm channels and send windows (Q3, U-7)
- [ ] Confirm the business impact of delayed or duplicate notifications

### 11.3 Engineering Owner
- [ ] Adding `event_id` and `schema_version` to the outbox payload is acceptable (P-1)
- [ ] Relay placement in appointment (O1) or the feed API ownership (O3) is acceptable
- [ ] A narrow patient contact-read contract is feasible (minimum fields)
- [ ] The POC plan T1–T7 is feasible; the effort estimate is provided
- [ ] Consumer-side mocks will be supplemented with contract tests (B-4)

### 11.4 Security/Privacy
- [ ] Service-identity design approved (BL-2); key and blast-radius impact assessed
- [ ] Events remain IDs-only; contact-data persistence and retention rules defined (Q6)
- [ ] Consent and contact preferences ownership agreed (U-7)
- [ ] No tokens or contact data in logs (`AGENTS.md`); TLS on all new paths
- [ ] No compliance claims are made from this design alone

### 11.5 Operations/SRE
- [ ] SLO owner and on-call owner named (BL-4)
- [ ] Monitors defined: outbox lag, oldest unpublished event, notification failures, DLQ/cursor lag (F9)
- [ ] Runbooks: replay, redrive, cursor reset, rollback (T7)
- [ ] Capacity on the existing nodes is confirmed (A-2); HPA/PDB policy decided (F8)
- [ ] Backup and restore implications (re-sends after restore) accepted

### 11.6 Domain/Clinical Safety
- [ ] Notification content is non-clinical (no diagnosis, notes or results) (`AGENTS.md`)
- [ ] The time-zone correctness of the appointment times in messages is verified
- [ ] Out-of-order or stale messages can't tell a patient a cancelled visit is confirmed (T3)
- [ ] The behaviour for vulnerable patients and preferred channels is reviewed (Q3, U-7)

## 12. Possible human outcomes (for the decision owner to choose; none is pre-selected)

| Outcome | Meaning | What happens next | ADR status after |
|---|---|---|---|
| **A. Approve** | The decision owner accepts the proposal **as written**: the foundation + the O1/O3 shortlist, with selection by POC | Assign BL owners and dates; run the POC T1–T7; return with results to finalise O1 vs O3 | Recorded by the decision owner (e.g. Accepted for the foundation and shortlist). **Not set by this pack** |
| **B. Reject** | The proposal isn't acceptable | Record the reasons; optionally restate the decision question or constraints (e.g. broker approval) and regenerate the options | Rejected (recorded by the decision owner) |
| **C. Defer** | Evidence or stakeholder input is insufficient (e.g. BL-1, U-5, U-7) | List the missing inputs with owners and dates; reconvene when they are supplied | Remains Proposed (Deferred) |
| **D. Return for changes** | The direction is acceptable but the artefacts need correction | List the required changes (e.g. re-weighting, additional options, corrected evidence); the coordinator revises and resubmits | Remains Proposed |

**Decision record (to be completed by humans only):**

| Field | Entry |
|---|---|
| Outcome (A/B/C/D) | ______ |
| Decision owner (name, role) | ______ |
| Date | ______ |
| Conditions / required changes | ______ |
| Reviewer sign-offs (role, name, date) | ______ |
