# Four-Axis Trade-off: Booking Notification Options

| Field | Value |
|---|---|
| Options assessed | **All six** in `option-set.md`: O0 do nothing, O1 sync push, O2 durable queue, O3 consumer-pull feed, O4 pub/sub backbone, O5 hybrid |
| Axes | 1 Latency, 2 Cost, 3 Consistency, 4 Operability. **None omitted** |
| Inputs | `decision-brief.md`, `current-state-architecture.md`, `evidence-register.md` (`E-nn`), `evidence-validation-report.md` (`V-nn`, B-1–B-6) |
| Prices / benchmarks | **None used or invented.** No measurement exists for any option (V-23 notes there are no load tests either) |

## 1. Rating scale

| Rating | Meaning |
|---|---|
| 5 | Strong: meets the axis concern with little added risk; supported by evidence |
| 4 | Good: minor compromises, manageable with known mitigations |
| 3 | Adequate: real compromises that need design effort or validation |
| 2 | Weak: significant compromises or risks |
| 1 | Poor: does not meet the axis concern |
| **Unknown** | The deciding input is missing (no target, price or measurement). **Unknown is never treated as 3 or better, and never counts towards a shortlist** |
| n/a | The axis does not apply because the option delivers no notifications (O0) |

The ratings are qualitative and relative within this option set.

## 2. Comparison matrix

| Option | Axis 1 Latency | Axis 2 Cost | Axis 3 Consistency | Axis 4 Operability |
|---|---|---|---|---|
| O0 Do nothing | Booking path 5; delivery **n/a** | 5 (no change), but delivers no outcome | n/a (no notification state) | 5 (nothing new) |
| O1 Sync push from in-service relay | Booking path 5; end-to-end **Unknown** | 4 | 4 | 4 |
| O2 Durable point-to-point queue | Booking path 5; end-to-end **Unknown** | **Unknown** (managed-service pricing) | 3 | 3 |
| O3 Consumer-pull change feed | Booking path 5; end-to-end **Unknown** (poll-interval bound) | 4 | 4 | 4 |
| O4 Pub/sub backbone | Booking path 5; end-to-end **Unknown** | **Unknown** (managed-service pricing) | 3 | 2 |
| O5 Hybrid | Booking path 5; end-to-end **Unknown** | **Unknown** | 3 | 2 |

The booking-path latency is 5 for every option for the same reason: the event row is already written in the booking
transaction (E-10, V-06), and every option processes it out of band. The axis therefore **does not differentiate**
the options on the booking path. End-to-end delivery latency is Unknown for all of them, because there's no target
(B-1) and no measurement.

## 3. Axis detail per option

### Axis 1 — Latency

| Option | Request path and hops (after commit) | Sync / async | Hot / cold path | Tail-latency effects | Measurements available | Validation required |
|---|---|---|---|---|---|---|
| O0 | none | — | — | — | — | — |
| O1 | relay poll → HTTP → notification → contact lookup (HTTP) → adapter: 3 network hops | async relay + 2 sync calls | Cold path (off booking) | Notification or patient slowness grows the backlog; retries add delay | None | Measure p50/p95 from booking commit to request stored under normal and degraded upstreams |
| O2 | relay poll → queue publish → consumer receive → contact lookup → adapter: 4 hops | async ×2 + 1 sync | Cold | Redelivery after the visibility timeout (60 s default, E-11) adds tail delay | None | Same measurement + redelivery tail |
| O3 | consumer poll (interval *t*) → feed read → contact lookup → adapter: 3 hops | async pull + 1 sync | Cold; feed reads hit appointment's DB | Latency floor ≈ poll interval; batch size affects tail | None | Measure with candidate intervals; DB read cost |
| O4 | relay → topic → routing → subscription → consumer → lookup → adapter: 5 hops | async ×3 + 1 sync | Cold | The most hops; each retry layer adds tail | None | Same + per-hop timing |
| O5 | O2/O3 path for class (a); scheduled batch for class (b) | mixed | Cold | Batch class latency = schedule interval by design | None | Per class |

Assumption A-1: a delay of minutes is acceptable (brief §5.4 **Open**). If the target is seconds, O3's poll interval
and O2/O4 redelivery tails become decisive.

### Axis 2 — Cost (no prices)

| Option | Compute | Storage | Network | Licence / managed service | Engineering effort | On-call burden | Scaling behaviour | Pricing inputs needed |
|---|---|---|---|---|---|---|---|---|
| O1 | Relay in existing appointment pods; notification pods | Notification store; outbox retention | Intra-cluster HTTP | None new | Medium: relay, retry, client, notification | Low–Medium | Scales with appointment pods; backpressure is hand-built | Pod sizing only |
| O2 | Relay + consumer pods | Queue retention + DLQ; notification store | Calls to the managed queue | **Managed queue (pricing Unknown)**; the resource is already defined in infrastructure code (E-11) | Medium | Medium (DLQ, redrive) | Queue absorbs bursts; consumers scale on depth | Per-request and retention pricing; quotas |
| O3 | Poller pods; extra reads on appointment DB | Outbox kept as a log (retention) | Intra-cluster HTTP | None new | Medium: feed API, cursor, ordering key | Low–Medium | Read load on single-AZ RDS (E-20) grows with consumers | RDS capacity headroom |
| O4 | Relay + per-subscriber consumers | Subscriptions + DLQs | Calls to managed backbone | **Managed backbone (pricing Unknown)** | High: schema governance, routing, subscriptions | High | Fan-out without producer change | Per-event and per-subscription pricing; quotas |
| O5 | Sum of two paths | Both | Both | Depends on class (a) path | High | High | Per class | Depends |

O1 and O3 are rated 4 under Assumption A-2: notification runs on the existing EKS nodes (2 spot t3-class; E-24).
**Node capacity for an added service hasn't been measured**, so this is Medium confidence.

### Axis 3 — Consistency

| Option | Source of truth | Read-after-write | Transaction boundary | Replication lag / staleness | Ordering | Concurrent updates / conflicts | Data correctness risks | Rating |
|---|---|---|---|---|---|---|---|---|
| O1 | Outbox → notification store | The notification store is consistent after a 2xx response | Booking tx (row) ‖ notification tx (request) | Contact data read at send time (fresh) | Relay order; lost on retry unless the consumer checks `version` | Dedupe on `event_id` as the idempotency key | Crash window between store and send (ARB H-1) | 4 |
| O2 | Outbox → queue → notification store | The store is consistent after ack | 3 boundaries | Fresh contact read | **Not guaranteed**; consumer `version` check | Duplicates on redelivery; dedupe required | Messages lost if they expire past retention (ARB H-3) | 3 |
| O3 | Outbox as log; cursor owned by the consumer | Consumer sees commits in sequence order | Booking tx ‖ consumer tx (cursor + request together) | Bounded by poll interval | **Possible** via a commit-ordered sequence (must be added) | Cursor + dedupe; replay by resetting the cursor | Gaps if the sequence isn't commit-ordered | 4 |
| O4 | Outbox → subscriptions → store | After ack | 3+ boundaries | Fresh contact read | **Not guaranteed** | Duplicates per hop | Schema drift across consumers (ARB M-9) | 3 |
| O5 | Per class | Per class | Two mechanisms | Batch class stale by schedule | Per class | Cross-class duplicates without a shared key | Two dedupe domains | 3 |

There is no cache in any option, so cache invalidation doesn't apply. Booking concurrency correctness stays with
appointment's GiST constraint + optimistic version in every option (E-21).

### Axis 4 — Operability

| Option | Deployment / upgrades | Monitoring / alerting / tracing | SLO ownership | Backup / restore / DR | Capacity | Security / patching | Runbooks | Skills / on-call | Rating |
|---|---|---|---|---|---|---|---|---|---|
| O1 | Same pipeline as the other services (Argo CD + Helm library chart, C9/C11) | Custom outbox-lag metric needed; Datadog APM covers HTTP (E-46) | **Unknown** owner | Outbox in RDS (7-day backups, single-AZ, E-20); restore can re-send (dedupe needed) | Relay shares appointment pods | Existing patterns (non-root, read-only FS; V-12) | New: relay backlog, redrive | Existing skills; on-call **Unknown** | 4 |
| O2 | + managed queue (infrastructure code exists, E-11) | Queue depth/age/DLQ metrics; **no monitors exist today** (V-22) | **Unknown** | Queue state isn't restored with RDS; DR across two stores | Managed | Queue policy must keep the TLS-only rule (ARB H-4) | New: DLQ redrive, retention | New skill area; on-call **Unknown** | 3 |
| O3 | Feed API in appointment + poller | Lag = head − cursor (easy); APM covers HTTP | **Unknown** | Log + cursor restore together only if in the same backup set; replay is simple | DB read load on single-AZ RDS | Feed authorization for service callers (P-3) | New: cursor reset/replay | Existing skills | 4 |
| O4 | + backbone, routing, subscriptions per consumer | Per-hop metrics; tracing across hops is harder | **Unknown** | Per-subscription state; archive/replay policy | Managed | Per-subscriber permissions; schema governance | Most runbooks | New skills; on-call **Unknown** | 2 |
| O5 | Two mechanisms | Both | **Unknown** | Both | Both | Both | Both | Both | 2 |

SLO and on-call ownership are Unknown for every option (brief §7, Q8). That is the same for all options and doesn't
change the relative ratings, but it blocks approval.

## 4. Narrative per option

- **O0:** cheapest and simplest, and fails the business outcome (brief §1.1). It is a baseline only.
- **O1:** no new platform component and existing skills. Its weaknesses are point-to-point coupling and hand-written
  retry and backpressure. Its consistency depends on the crash-window fix (ARB H-1).
- **O2:** buffering and dead-lettering come from the platform. Its weaknesses are no ordering, a redelivery tail, an
  unknown managed-service cost and a new operational surface. It needs broker approval.
- **O3:** ordering and replay are natural. Its weaknesses are a polling-interval latency floor and read load on the
  single-AZ RDS. It needs no broker.
- **O4:** the best fan-out. It has the most hops and the highest governance and operational load, and isn't justified
  for one consumer unless Q2 shows more consumers.
- **O5:** only relevant if Q2/Q3 confirm distinct workload classes. It is the highest operability cost.

## 5. Assumptions and unknowns register

| ID | Type | Statement | Affects | Owner (brief) |
|---|---|---|---|---|
| A-1 | Assumption | A delay of minutes is acceptable | Axis 1 all options | Product owner (Q4) |
| A-2 | Assumption | Notification runs on existing nodes without new capacity | Axis 2 O1/O3 | SRE (Q8) |
| A-3 | Assumption | Demo-scale volume | Axes 1–4 | Product owner (Q4) |
| U-1 | Unknown | End-to-end latency target and measurements | Axis 1 | B-1 |
| U-2 | Unknown | Managed queue and backbone pricing and quotas | Axis 2 O2/O4/O5 | Provider validation |
| U-3 | Unknown | Whether fan-out or other consumers are required | Axes 2–4 O4 | Q2 |
| U-4 | Unknown | SLO and on-call owners | Axis 4 all | Q8, §7 |
| U-5 | Unknown | Whether a broker is approved | O2/O4 feasibility | Governance (`AGENTS.md:35,60`) |
| U-6 | Unknown | RPO/RTO | Axes 3–4 | Q5 |

## 6. Sensitivity analysis (qualitative)

| If… | Effect on the comparison |
|---|---|
| The latency target is seconds, not minutes (U-1) | O3 drops (poll floor); O1 rises; O2/O4 depend on the measured tail |
| More consumers are confirmed (U-3) | O4 rises (fan-out); O1 drops (coupling); O3 holds (each consumer pulls) |
| A broker is **not** approved (U-5) | O2/O4 (and O5 if class (a) uses a queue) become infeasible; O1 and O3 remain |
| Managed pricing is low relative to budget (U-2) | O2's cost rating could reach 4; O4 stays limited by operability |
| Strict ordering per appointment is required (D4) | O3 is favoured; O2/O4 need extra design (ordered/partitioned delivery) |
| RPO/RTO is strict (U-6) | Options with one restore set (O1, O3) are favoured over multi-store options |

## 7. Shortlist and recommendation

**Shortlist (evidence-based):**
- **O1 and O3** are the only options with no Unknown on axes 2–4 and no broker-approval dependency.
- **O2** stays shortlisted **conditionally**, pending U-2 (pricing) and U-5 (broker approval).
- O4 and O5 are **not shortlisted** until Q2 confirms fan-out or distinct workload classes.

**Recommendation: none yet. The evidence is insufficient.**
- Axis 1 is Unknown for every option (no target, no measurement).
- Axis 2 is Unknown for O2, O4 and O5.
- SLO and on-call ownership are Unknown for all options.

Choosing now would be premature.

**Validation needed before a recommendation:**
1. Agree the B-1 targets and the driver weights.
2. Run a POC of O1 and O3, and O2 if a broker is approved. Measure booking-commit → notification-request latency
   (p50/p95/p99) under normal conditions, with upstream degraded, and with a consumer outage. Also measure duplicate
   and loss counts after fault injection.
3. Get provider pricing and quotas for O2 at the agreed volume.
4. Name the SLO and on-call owners.
