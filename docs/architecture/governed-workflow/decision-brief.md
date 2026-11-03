# Decision Brief: Booking Notification Communication

| Field | Value |
|---|---|
| Brief ID | DB-001 |
| Decision topic | How appointment-service communicates booking outcomes so patients can be notified (the same topic as ADR-001) — **Open: confirm with the decision owner** |
| Stage | Pre-discovery decision brief (governed workflow step 0). **No technology is proposed here** |
| Status | Draft — all **Open** items need stakeholder input before discovery starts |
| Related | `docs/architecture/cross-repository-discovery.md`, `dependency-mapping.md`, `architecture-options-tradeoff.md`, `ADR-001-notification-architecture.md` (Proposed), `architecture-pack-review.md` (on unmerged branches; see §9) |

**Labels:**
- **Known**: stated in a repository file (cited as `repo:path`).
- **Assumption**: accepted for planning, not verified.
- **Open**: needs stakeholder input.

Nothing here selects a protocol, broker, cloud service or product.

---

## 1. Business outcome and user impact

| # | Item | Status |
|---|---|---|
| 1.1 | Patients are told when an appointment is booked, so they know when and where to attend | Known need (problem statement given for ADR-001); the business owner hasn't restated it — **Open** |
| 1.2 | Booking must keep working when notification is unavailable: no booking is refused or delayed because of notification | Known need (same problem statement) |
| 1.3 | Whether rescheduling and cancellation also notify the patient | **Open** (the events exist: `appointment-service:app/domain/appointments.py`; whether to notify on them is not decided) |
| 1.4 | User groups affected: patients (recipients); receptionists who book (`appointment-service:README.md` roles); support staff handling failed notifications | Patients and receptionists Known; support role **Open** |
| 1.5 | Impact of failure: a missed notification (patient doesn't attend), a duplicate (confusion, loss of trust), or wrong content (privacy incident) | Assumption (no incident data exists) |
| 1.6 | Business value measure (e.g. no-show reduction) | **Open**: no baseline or target in any repository |

## 2. System and repository scope

| Repository (`UST-HealthCare-Project` org) | Role in this decision | Status |
|---|---|---|
| `appointment-service` | Producer of booking outcomes; owns `appointments` and `outbox_events` | Known (`app/services/appointment_service.py:188-230`) |
| `patient-service` | Owner of patient contact data (`phone`, `email`) | Known (`app/schemas/patient.py:41-49,95-105`) |
| `identity-service` | Issuer of tokens; any service identity for background processing | Known (`app/core/tokens.py`) |
| *(notification capability)* | Planned Phase 8 owner of notification requests, templates and delivery status | Known as planned (`smart-hospital-architecture:AGENTS.md:28,35`); **no repository or code** |
| `smart-hospital-infra` | Any platform resources the decision needs | Known (`infra/terraform/`) |
| `smart-hospital-gitops` | Deployment of any new or changed workload | Known (`deploy/`) |
| `smart-hospital-architecture` | Governance, ADRs, this brief | Known |
| Monorepo `Msocial123/UST-HealthCare-Project` | Currently the live deployment source (Argo CD follows its `main`) | Known (`deploy/argocd/apps/values.yaml:1` in the monorepo) |

## 3. In scope and out of scope

**In scope:**
- How booking outcomes are communicated from appointment-service to the notification capability.
- The delivery guarantees, failure isolation and recovery for that path.
- Duplicate and ordering behaviour.
- Contact-data access and privacy.
- Observability and operations for the path.
- Migration, rollout and rollback.

**Out of scope (unless the owner changes this):**

| Item | Reason |
|---|---|
| Choice of a specific broker, cloud service or protocol | Decided later, after discovery and trade-off (this brief must not propose one) |
| Notification channel providers (real email/SMS vendors) | `AGENTS.md:60` requires explicit approval for real email/SMS |
| Clinical content, reminders that carry clinical advice | `Smart_Hospital_Repository_Rules.md:34`; `AGENTS.md:45` |
| Encounter / visit service | Separate phase (`AGENTS.md:27`) |
| Frontend, API gateway | Future scope (`AGENTS.md:37`) |
| Changing the booking concurrency model | Appointment's integrity design is not part of this decision (`AGENTS.md:34`) |
| Patient consent and preference management | **Open**: may need to come into scope (§8) |

## 4. Constraints and dependencies

| # | Constraint / dependency | Evidence | Status |
|---|---|---|---|
| 4.1 | Each service owns its data; no cross-service DB access | `AGENTS.md:30-31` | Known |
| 4.2 | Notification must be transport-independent, with a mock adapter and a durable request/status model first; a broker only when explicitly requested and justified | `AGENTS.md:35`; `Smart_Hospital_Repository_Rules.md:50` | Known |
| 4.3 | Notification content must be generic and non-clinical; minimum data | `Smart_Hospital_Repository_Rules.md:34`; `AGENTS.md:45` | Known |
| 4.4 | Idempotency is required for repeatable notification processing | `Smart_Hospital_Repository_Rules.md:49` | Known |
| 4.5 | Synthetic data only in dev and test | `AGENTS.md:39` | Known |
| 4.6 | appointment already writes booking events in the same transaction as the booking (outbox), and nothing reads them | `appointment-service:app/services/appointment_service.py:188-230`; `app/models/outbox_event.py` | Known |
| 4.7 | appointment holds only `patient_id`; contact details live in patient-service | `appointment-service:app/models/appointment.py:21,49` | Known |
| 4.8 | Services today call each other with the end user's token; there is no service identity for background work | `scheduling-service:app/clients/facility_client.py:66-70` | Known; resolution **Open** |
| 4.9 | One shared token-signing key across services | `smart-hospital-infra:modules/secrets/main.tf:34-49` | Known |
| 4.10 | Platform: AWS account `165742853391`, `us-east-1`, EKS, RDS PostgreSQL 16 (single-AZ), spot nodes | `smart-hospital-infra:.../dev/terraform.tfvars` | Known (config); live state must be re-checked at discovery time |
| 4.11 | No custom domain in dev, so the email-identity resources in Terraform aren't created | `dev/main.tf:134-136`; `dev/terraform.tfvars:111` | Known |
| 4.12 | Budget, team size and support hours | — | **Open** |
| 4.13 | Regulatory or contractual obligations for patient communication | Repositories make no compliance claim (`AGENTS.md:46`) | **Open** |

## 5. Known requirements and measurable targets

| # | Requirement | Measurable target | Status |
|---|---|---|---|
| 5.1 | Booking succeeds independently of notification | A booking's success rate and latency are unaffected by notification being down | Requirement Known; **numeric tolerance Open** |
| 5.2 | No committed booking outcome is silently lost | Every event is either delivered or recorded in a reviewable failure store | Requirement Known (problem statement); **loss tolerance Open** |
| 5.3 | No duplicate patient messages | Duplicates per N notifications | **Open** |
| 5.4 | Time from booking to notification | p50 / p95 latency target | **Open** (no target in any repository) |
| 5.5 | Volume | Bookings per hour/day, peak factor | **Open** (no volume data in any repository) |
| 5.6 | Availability of the notification path | Availability objective | **Open** |
| 5.7 | Recovery | RPO / RTO for booking outcomes and notification state | **Open** |
| 5.8 | Retention | How long notification records and failed items are kept | **Open** |
| 5.9 | Channels | Email, SMS, both, other | **Open** |
| 5.10 | Send window / timezone rules | — | **Open** (facility timezone exists: `facility-service:app/models/facility.py`) |

No target in this table has been invented. Every number must come from the business owner or measurement.

## 6. Decision drivers and non-functional requirements

Names only. Weights are **Open** and must be agreed before options are scored.

| # | Driver / NFR | Why it matters | Source |
|---|---|---|---|
| D1 | Booking isolation | Booking must not fail because of notification | §1.2 |
| D2 | Delivery reliability (no silent loss) | Patients rely on the message | §5.2 |
| D3 | Duplicate safety and idempotency | Rule 4.4 | `Smart_Hospital_Repository_Rules.md:49` |
| D4 | Ordering per appointment | A cancel must not be processed before its create | Assumption, to confirm |
| D5 | Privacy and data minimisation | Rules 4.3 and 4.7 | `AGENTS.md:45` |
| D6 | Security (service identity, least privilege) | Constraint 4.8 / 4.9 | Known gap |
| D7 | Observability (can we see lag, failures, backlog?) | Operations | Datadog is the monitoring standard (`smart-hospital-infra:infra/terraform/README.md`) |
| D8 | Operability for the current team | Team size **Open** | §4.12 |
| D9 | Cost | Budget **Open** | §4.12 |
| D10 | Portability / lock-in tolerance | **Open** | — |
| D11 | Maintainability and testability (mockable, independently testable services) | `AGENTS.md:31` | Known |
| D12 | Recoverability (replay, redrive, restore) | §5.7 | **Open** targets |

## 7. Decision owner and required reviewers

| Role | Responsibility | Named person |
|---|---|---|
| **Decision owner**: accountable / principal architect | Approves the final ADR | **Open** |
| Business / product owner | Outcomes, channels, targets (§1, §5) | **Open** |
| Security & privacy reviewer | Service identity, contact-data handling, keys | **Open** |
| appointment-service owner | Producer contract and booking isolation | **Open** |
| patient-service owner | Contact-data contract, consent | **Open** |
| Platform / SRE | Infrastructure, operations, DR, cost | **Open** |
| Governance body / ARB | Gate approval (`docs/workflow/governed-sdlc.md`) | **Open** |

## 8. Open questions and missing stakeholder inputs

| # | Question | Needed from | Blocks |
|---|---|---|---|
| Q1 | Confirm the decision topic and that this brief replaces or feeds ADR-001 | Decision owner | Everything |
| Q2 | Which events trigger a notification (create only, or reschedule and cancel too)? | Product owner | Scope |
| Q3 | Channels and send-window rules | Product owner | §5.9, §5.10 |
| Q4 | Latency, volume, availability, duplicate and loss tolerances | Product owner + SRE | §5 |
| Q5 | RPO / RTO and retention | Product owner + SRE | §5.7, §5.8 |
| Q6 | Where do patient consent and contact preferences live, and must notification honour opt-outs? | patient-service owner + privacy | §3, D5 |
| Q7 | How does a background process authenticate to patient-service? | Security | D6 |
| Q8 | Budget and support model (who is on call for failed notifications?) | Sponsor + SRE | D8, D9 |
| Q9 | Is cloud-provider lock-in acceptable? | Governance body | D10 |
| Q10 | Any regulatory or contractual communication obligations? | Compliance / legal | §4.13 |
| Q11 | Is dev the reference environment for evidence, and will it stay up during discovery? | SRE | §9 |
| Q12 | Names for every role in §7 | Sponsor | Approval |

## 9. Evidence sources to inspect during discovery

| Source | What to confirm |
|---|---|
| `appointment-service` (`app/services/appointment_service.py`, `app/models/outbox_event.py`, `app/domain/appointments.py`, `tests/`) | Event types, payload contract (event id, version), transaction boundary |
| `patient-service` (`app/schemas/patient.py`, `app/api/v1/patients.py`, `app/domain/roles.py`) | Contact fields, access permissions, anything about consent |
| `identity-service` (`app/core/tokens.py`, `app/domain/roles.py`) | Options for service identity |
| `smart-hospital-infra` (`environments/dev/*.tf`, `modules/sqs`, `modules/ses`, `modules/secrets`, `modules/datadog-observability`) | What platform resources are defined, and monitoring coverage |
| `smart-hospital-gitops` (`deploy/helm/*`, `deploy/argocd/*`) | How a new workload would be deployed and configured |
| `smart-hospital-architecture` (`AGENTS.md`, `Smart_Hospital_Repository_Rules.md`, `docs/workflow/governed-sdlc.md`) | Governing rules and gates |
| Existing architecture pack (discovery, dependency map, trade-off, ADR-001, ARB review): unmerged branches `devin/1793638443-cross-repo-discovery`, `devin/1793641500-dependency-mapping`, `devin/1793642500-notification-options`, `devin/1793643500-adr-001-notification`, `devin/1793644500-architecture-pack-review` | Reuse the findings; address the ARB Critical/High items |
| Live AWS / EKS / Argo CD / Datadog state (read-only) | Which defined resources actually exist; current telemetry |
| Jira (KAN / UXBA), qTest project 32916 | Any stated notification requirements or test cases (**not yet inspected**) |
| Stakeholder interviews | Every item marked Open |

## 10. Definition of success

This decision is successful when:

1. The decision topic, scope and owner are confirmed (Q1, Q12).
2. Every target in §5 has an agreed value or is explicitly marked "not required" by the business owner.
3. Decision drivers in §6 have agreed weights **before** any option is scored.
4. Discovery covers every source in §9, separating facts, assumptions and unknowns.
5. The resulting ADR shows how each requirement in §5 and each driver in §6 is met, with measured POC evidence where targets are numeric.
6. The security and privacy reviewers sign off on service identity and contact-data handling (Q6, Q7).
7. The ARB / governance body approves through the gates in `docs/workflow/governed-sdlc.md`.
8. After implementation, the agreed targets are met in the reference environment and observed in monitoring.

**Status of this brief: Draft. 12 open questions (§8), every numeric target Open (§5), every reviewer name Open (§7).**
