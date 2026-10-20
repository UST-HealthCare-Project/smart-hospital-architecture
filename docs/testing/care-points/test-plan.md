# Care Points — Test plan and test design

**Status:** design only. The Care Points feature is not implemented yet. This document doesn't change any code or tests, and every case is expected to fail until the feature is built.

- **Service:** `services/appointment-service`. Care Points extends it with a fee per booking and a points ledger owned by this service.
- **Data:** synthetic only.
- **Nature:** an administrative loyalty scheme, not clinical logic. 1 point = ₹1.

**Related files:**
- [`test-cases.csv`](test-cases.csv): the full traced case set, revised after self-review. It is authoritative where it differs from Part B.
- [`traceability.md`](traceability.md): maps each AC to its case IDs.

## Requirement

As a receptionist, I can let a patient pay part of the consultation fee with Care Points when booking an appointment, so that loyal patients get a discount, without the points scheme ever affecting the clinical appointment itself.

**Test fee schedule** (synthetic, set in test config): ₹0, ₹190, ₹200, ₹500, ₹510, ₹10,000.

| AC | Rule | Criterion |
|---|---|---|
| AC-1 | Redeem at booking | Only when booking: request may include `points_to_redeem`. Response shows `consultation_fee`, `points_redeemed`, `net_fee = consultation_fee - points_redeemed`. Omitted or 0 = no redemption. |
| AC-2 | Minimum | A redemption, when requested, is ≥100 points. 1–99 → 422 `points_below_minimum`. |
| AC-3 | Step | Multiples of 10; otherwise 422 `invalid_points_step`. Negative or non-integer → 422. |
| AC-4 | Maximum | At most 50% of fee, rounded down to a multiple of 10. Above → 422 `exceeds_max_redemption`. |
| AC-5 | Balance | Cannot exceed available (unexpired) balance: 409 `insufficient_points`, no appointment, balance unchanged. No ledger = balance 0. |
| AC-6 | Zero fee | Fee 0 + any redemption → 422 `redemption_not_allowed`; booking without points works. |
| AC-7 | Atomic | Debit and booking commit in one transaction; booking failure (conflict, outbox failure, flag off) → no debit. |
| AC-8 | Cancel refund | Cancel ≥24h before `starts_at` → full refund; <24h → forfeit. Exactly 24h00m00s refunds. No zero-value ledger entries. |
| AC-9 | Reschedule | Keeps fee and redemption unchanged; refund rule uses the new `starts_at` at cancellation time. |
| AC-10 | No-show/complete/earn | No-show forfeits redeemed points, earns nothing. COMPLETED earns 5% of `net_fee`, rounded down; net 0 earns nothing. Booking alone earns nothing. |
| AC-11 | No clinical effect | Points never change slot availability, double-booking, transitions or scheduling advice. |
| AC-12 | Roles/min data | `GET /api/v1/patients/{id}/points`: RECEPTIONIST, ADMIN only; DOCTOR/READ_ONLY 403; no token 401. DOCTOR appointment responses omit fee and points fields. |
| AC-13 | Concurrency | Two simultaneous redemptions on one balance never overspend; balance never negative. |
| AC-14 | Expiry/order | Points expire 365 days after earning (day 365 valid, day 366 expired). Redemption consumes oldest unexpired first. |
| AC-15 | Data minimisation | Logs and outbox events never contain balances or fee amounts. |
| AC-16 | Flag | `POINTS_ENABLED` defaults false. False + `points_to_redeem > 0` → 503 `feature_disabled`, no debit; booking without points works. |
| AC-17 | Migration | Ledger migration additive, with working downgrade. |

---

# Part A — Test plan

## A1. Scope

**In scope:**
- **Booking:** `points_to_redeem` on `POST /api/v1/appointments`; the fee fields `consultation_fee`, `points_redeemed` and `net_fee`.
- **Redemption rules:**
  - minimum 100 (AC-2)
  - multiples of 10 (AC-3)
  - 50% cap (AC-4)
  - available balance (AC-5)
  - zero fee (AC-6)
- **Ledger effects:**
  - atomic debit with the booking (AC-7)
  - refund or forfeit on cancel, including after reschedule (AC-8, AC-9)
  - forfeit on no-show and earning on completion (AC-10)
  - 365-day expiry and oldest-points-first redemption (AC-14)
- **Isolation from the clinical appointment (AC-11):** the existing booking rules must behave identically with and without points.
- **Roles:** `GET /api/v1/patients/{id}/points`, and fee fields removed from DOCTOR responses (AC-12).
- **Concurrency:** no overspend under simultaneous redemptions (AC-13).
- **Data minimisation:** no balances or fees in logs or outbox events (AC-15).
- **Feature flag:** `POINTS_ENABLED` defaults to off (AC-16).
- **Migration:** the ledger migration is additive and downgrades (AC-17).

**Out of scope:**
- Patient, Identity, Scheduling or Notification services; patient IDs are not validated against them, same as today.
- Payment processing, real currency, invoices, tax.
- Points awarded manually or by admins, transfers, or expiry notifications. None of these are in the spec.
- Publishing outbox events (no broker yet).
- Load and performance testing beyond the AC-13 race tests.
- UI.
- Any clinical logic.
- Changing application code or existing tests during design.

## A2. Test items

| Item | Notes |
|---|---|
| Booking schema and service | New `points_to_redeem` field; validation order; fee lookup (A1: fee comes from a test-config mapping) |
| Points domain rules | Pure functions: cap, minimum, step, earning (5% rounded down), expiry, consumption order |
| Points ledger repository and model | Lots or entries, balance query, FIFO debit, refund. No zero-value rows. |
| Cancel / no-show / complete service paths | Ledger side effects |
| `GET /api/v1/patients/{id}/points` | New route, new permission |
| Response serialisation | Fee fields removed depending on the caller's role |
| Settings | `points_enabled` (default False), `.env.example` |
| Alembic migration | New ledger table(s); downgrade |
| Logging and outbox | Payload key set unchanged; no amounts |

## A3. Approach and test levels

| Level | What | Technique | DB | Main CP-* cases |
|---|---|---|---|---|
| **Unit (L1)** | Cap / minimum / step / earning / expiry / FIFO as pure functions; role → field visibility; settings validation | Boundary values per fee, equivalence classes, decision table | none | CP-2/3/4 cap math, CP-10 rounding, CP-14 expiry, CP-INT-7 sweep |
| **API (L3)** | Full HTTP contract: status, error code, envelope, response fields, ledger and balance end state, no side effects on rejection | Boundary tables, refund decision table, the proposed check order (Q6), fault injection | SQLite and PG | CP-2..CP-14 tables, CP-DT-*, CP-INT-*, CP-NEG-1..22, 27..36 |
| **PostgreSQL (L4)** | Concurrent redemptions and cancels; DB-level non-negative balance guarantee | `threading.Barrier` after the balance read; N=2 and N=10; assert final ledger sums | PG only | CP-NEG-23..26 |
| **Migration (L6)** | upgrade → downgrade −1 → upgrade → downgrade base on SQLite and PG; additive only (existing `appointments` columns unchanged, or new nullable/defaulted ones); `alembic check` clean | Schema inspection | SQLite and PG | AC-17 |

Rules:
- **AC-11 regression:** re-run the existing booking-integrity tests parametrised with `points_to_redeem ∈ {omitted, 100}`, expecting identical results.
- **Atomicity (AC-7):** after every rejected or failed request, assert three things: no appointment created, balance unchanged, no new ledger or outbox row.
- **Clock-driven rules (AC-8, AC-14):** move the clock by setting `app.state.clock`; never use real sleeps.
- **Not-yet-specified behaviour:** pin it as characterisation once Q1–Q9 are answered.

## A4. Environments and test data

| Environment | Use |
|---|---|
| SQLite in-memory (`make_settings()` default) | Fast feedback for L1/L3/L6. **No concurrency claims.** |
| PostgreSQL 14+, disposable DB via `TEST_DATABASE_URL` | Whole suite, plus L4 races and the PG migration round trip. Required for CP-NEG-23..26 and AC-13 sign-off. |
| CI (`.github/workflows/ci.yml`) | Ruff, SQLite pytest, PG pytest, Alembic round trip and check, Gitleaks. The PG job must stay mandatory. |

**Test data (all synthetic):**

| Data | Values |
|---|---|
| Fee schedule (test config) | ₹0, ₹190, ₹200, ₹500, ₹510, ₹10,000 |
| Cap per fee | 0 / 90 / 100 / 250 / 250 / 5000 |
| Proposed additions (gap G-A) | ₹19 and ₹20 fees, to hit the first earning threshold (net ₹20 → 1 point) |
| Balances | 0 (no ledger), 99, 100, 490, 500, 10,000 |
| Lots | earned at `FIXED_NOW − n days` (n = 1, 10, 300, 364, 365, 366, 400); leap-year lot earned 2031-03-01 |
| Points values | 0, 1, 10, 90, 95, 99, 100, 101, 105, 109, 110, 240, 250, 255, 260, 4990, 4999, 5000, 5010, −10, 100.5, "100", null, 10,000,000 |
| Times | `FIXED_NOW` = 2030-01-15 08:00Z; `starts_at` via `at()` (March 2030); cancel offsets 23:59:59, 23:59:59.999999, 24:00:00, 24:00:01, 0, −1 s |
| Identities | Random UUIDs for patient, provider, facility; tokens from `mint_token()` |

## A5. Entry and exit criteria

**Entry:**
1. Answers on Q1–Q9, or the stated defaults accepted in writing.
2. A1 decided: how a booking gets its fee.
3. Feature branch implements AC-1..AC-17 behind `POINTS_ENABLED`.
4. Ledger migration present.
5. Existing suite green on SQLite and PG before the new tests are added (baseline: 171 passed + 6 skipped on SQLite, 177 passed on PG).
6. A disposable PG database is available.

**Exit:**
| # | Criterion |
|---|---|
| X1 | Every AC has at least one passing test at its required level. AC-13 must pass on PG; AC-17 must pass on SQLite and PG. |
| X2 | All P1 boundary, decision-table and interaction cases pass. Any deferral is listed with a reason. |
| X3 | SQLite: 0 failures; only PG-only tests skipped, and each is listed. PG: 0 failures, **0 skips**. |
| X4 | PG concurrency cases pass 20 runs in a row; no barrier timeouts; the balance is never negative in any run. |
| X5 | AC-11 regression set identical with and without points. |
| X6 | Existing tests unchanged, except for documented response-shape updates (see §7). No secrets; synthetic data only. |
| X7 | `ruff check`, `ruff format --check`, Alembic upgrade/downgrade/upgrade plus `alembic check` all clean; CI green. |
| X8 | Results reported as PASS / FAIL / NOT RUN with the actual commands run (AGENTS.md completion report). Failing cases are filed as findings. |

**Suspend:** PG unavailable (L4 can't be claimed); more than one flake in 20 concurrency runs; any P1 atomicity failure (points debited without a booking).

## A6. Risks

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| K1 | Overspend under concurrency (debit read-modify-write race) | Negative balance, financial error | L4 barrier tests; DB-level guard (row lock, `CHECK balance ≥ 0`, or an atomic conditional update) asserted on PG |
| K2 | Debit committed when the booking fails | Points lost | Fault injection at the ledger and outbox writes, plus conflict and flag-off cases (CP-NEG-20..22, 12, 15) |
| K3 | Points change clinical behaviour | Breaks the AC-11 guarantee | Parametrise the existing integrity tests with points |
| K4 | Off-by-one at 24h, day 365/366, rounding | Wrong refund, expiry or earn | Exact-instant boundaries via the injected clock; ambiguity Q4 resolved first |
| K5 | Unspecified check order (Q6) | Flaky expectations, contested defects | Pin the order once decided; decision-table row per conflict |
| K6 | Fee or balance leaking into logs or outbox | Breaks AC-15 / AGENTS.md | Extend the existing log-privacy and outbox key-set tests |
| K7 | DOCTOR response leaks fee fields | Breaks AC-12 | Exact key-set assertions per role |
| K8 | SQLite gives false confidence on locking | Race bugs missed | No concurrency sign-off from SQLite; PG CI job mandatory |
| K9 | Fee schedule cannot reach net ₹19/₹20 | Earning threshold untested | Add ₹19 and ₹20 to the test config (G-A) |
| K10 | Destructive migration | Data loss on downgrade | Additive-only check; downgrade drops only the new tables |

## A7. Existing-asset reuse summary

| Existing asset | Reused for |
|---|---|
| `make_settings(**overrides)` | `points_enabled=True/False`, fee mapping overrides |
| `build_app` / `teardown_app` | Apps built with the flag off at start; apps with a custom fee config |
| `app.state.clock` (`FIXED_NOW`) | AC-8 cancel offsets, AC-14 expiry days. Change the clock between book and cancel. |
| `booking_payload` / `book` | Add `points_to_redeem=`; `book` stays the standard arrange step |
| `mint_token` / `bearer` / `auth(role)` | Role cases for the points endpoint and DOCTOR field removal |
| `ENDPOINTS` / `FORBIDDEN` / `ALLOWED` matrix | Add `("get", "/api/v1/patients/{id}/points", {ADMIN, RECEPTIONIST})`; the 401/403 cases then extend automatically |
| PG `_race` + `Barrier` monkeypatch | AC-13: barrier after the balance read (wrap the new ledger-repository read the same way `AppointmentRepository.get` is wrapped) |
| Outbox failure helper (`add_event` patch + `raise_server_exceptions=False`) | AC-7 rollback of the debit; the same pattern patches the ledger write |
| `_outbox(app)` | AC-15: outbox payload key set stays exactly the current 10 keys |
| `_rendered(caplog)` | AC-15: assert balances and fees are absent from logs |
| `FakeSchedulingClient` | AC-11: advice behaves identically with points |
| `_round_trip` / `_alembic_config` | AC-17: add the new ledger tables to `TABLES` |
| New (proposed) | `points_lot(patient_id, amount, earned_at)` seeding fixture; `balance(app, patient_id)` helper; fee-mapping test config |

**Existing tests that will need deliberate updates when the feature lands:**
- `test_get_appointment`: compares a DOCTOR's GET with the ADMIN create response, which will fail once the fee fields are removed for DOCTOR.
- `test_invalid_booking_is_422[extra]` and `test_created_by_comes_from_the_token_subject`: still valid. `points_to_redeem` becomes a known field, but other unknown fields keep returning 422.

## A8. AC → existing tests that already cover part of it

| AC | Existing tests (partial coverage) | What's reused / still missing |
|---|---|---|
| AC-1 | test_book_appointment; test_invalid_booking_is_422[extra]; test_created_by_comes_from_the_token_subject | Booking response shape and unknown-field rejection exist; fee fields and `points_to_redeem` are missing |
| AC-2 | test_invalid_booking_is_422; test_validation_errors_do_not_echo_submitted_values | 422 envelope and no echoing of input; minimum rule missing |
| AC-3 | test_invalid_booking_is_422 (`bad-id`, `extra`) | Type / format rejection pattern; step and integer rules missing |
| AC-4 | test_invalid_booking_is_422 | Envelope only; cap rule missing |
| AC-5 | test_scheduling_unavailable_time_is_409_and_not_stored; test_conflicting_booking_writes_no_outbox_event | "409 and nothing stored" pattern; balance missing |
| AC-6 | test_book_appointment | Booking without points; zero-fee rule missing |
| AC-7 | test_outbox_failure_rolls_back_the_booking; test_exclusion_constraint_is_final_authority_when_precheck_is_bypassed; test_conflicting_booking_writes_no_outbox_event; test_flag_off_returns_documented_error_for_book_reschedule_cancel | Transaction rollback on outbox failure, conflict and flag off; the debit is not covered |
| AC-8 | test_cancel_frees_the_slot; test_cancel_with_stale_version_or_bad_reason; test_invalid_status_transition_is_409 (cancel from CHECKED_IN / COMPLETED / CANCELLED); test_concurrent_cancels_with_same_version_one_wins | Cancel lifecycle and version handling; refund and 24h rule missing |
| AC-9 | test_reschedule; test_reschedule_with_stale_version_is_409; test_outbox_events_are_written_with_each_change | Reschedule mechanics; fee unchanged and refund against the new start missing |
| AC-10 | test_check_in_then_complete; test_no_show_frees_the_slot; test_status_transitions_do_not_emit_events; test_invalid_status_transition_is_409 | Transitions; forfeit and earning missing |
| AC-11 | test_overlapping_booking_for_same_provider_is_409; test_adjacent_and_other_provider_bookings_are_allowed; test_completed_appointment_still_occupies_the_slot; test_database_stays_final_authority_when_scheduling_says_available; test_booking_consults_scheduling_when_enabled; test_simultaneous_bookings_for_same_slot_one_wins | The full clinical rule set to re-run with points |
| AC-12 | test_wrong_role_is_403; test_allowed_roles_pass_authorization; test_missing_token_is_401; test_invalid_tokens_are_401; test_expired_token_is_401_on_every_endpoint; test_caller_supplied_roles_are_ignored; test_get_appointment | Role matrix machinery; points route and DOCTOR field removal missing |
| AC-13 | test_simultaneous_bookings_for_same_slot_one_wins; test_exclusion_constraint_is_final_authority_when_precheck_is_bypassed; test_concurrent_cancels_with_same_version_one_wins | Race harness; balance races missing |
| AC-14 | test_booking_in_the_past_is_422 (clock-driven rule) | Fixed-clock pattern; expiry and oldest-first redemption missing |
| AC-15 | test_requests_never_log_tokens_or_request_values; test_request_log_uses_endpoint_name_not_raw_path; test_outbox_events_are_written_with_each_change (exact payload key set); test_json_formatter_outputs_structured_fields | Privacy harness; add fee and balance values to the forbidden-strings list |
| AC-16 | test_flag_off_returns_documented_error_for_book_reschedule_cancel; test_flag_off_keeps_reads_and_status_transitions_working; test_authentication_and_authorization_run_before_the_flag; test_flag_back_on_restores_booking; test_valid_configuration_defaults; test_env_example_has_flag_off_and_placeholders_only; test_invalid_configuration_fails_fast[appointments_enabled] | The same pattern for `POINTS_ENABLED`; points flag missing |
| AC-17 | test_migrations_upgrade_and_downgrade_sqlite; test_migrations_upgrade_and_downgrade_postgresql | Round-trip harness; ledger tables missing |

No AC is fully covered today (the feature does not exist). Every AC has reusable partial coverage; AC-6 and AC-14 have the least.

---

# Part B — Boundary tables, refund decision table, interactions and negative paths

These tables are the source design. The self-review revised some of their rows, and those revisions are in `test-cases.csv`:
- `starts_at` is now relative to the moved clock, so the future-start rule can't mask an expiry case.
- Barriers are placed before ledger access, so a correct design that locks the balance row doesn't deadlock the test.
- Duplicate rows were removed.

Where Part B and the CSV differ, the CSV wins.

Case IDs: `CP-<AC>-<n>`. Expected codes are those in AC-1..AC-17; where the AC gives none, `validation_error` (existing envelope) is assumed.

## B0. Fixed test conditions

| Item | Value |
|---|---|
| Fee schedule (test config) | ₹0, ₹190, ₹200, ₹500, ₹510, ₹10,000. **Assumption A1:** fee is chosen through a test-config mapping (e.g. by `reason_code` `FEE_0`…`FEE_10000`); the spec doesn't say how a booking gets its fee. |
| Max redemption per fee (AC-4: `floor(fee × 0.5 / 10) × 10`) | 0→0 · 190→**90** · 200→**100** · 500→250 · 510→**250** (255 rounded down) · 10000→5000 |
| Balance seeding | Fixture `points_lot(patient_id, amount, earned_at)` inserts ledger lots directly (synthetic). Default "ample" balance = 10,000, earned at `FIXED_NOW − 1 day`. |
| Clock | `app.state.clock`, starting at `FIXED_NOW` = 2030-01-15 08:00Z; tests move it for AC-8/AC-14. |
| Flags | `appointments_enabled=True`, `points_enabled=True` unless the case says otherwise. |
| Isolation rule | Each row picks fee and balance so that **only the rule under test can fail**. "Isolating fee/balance" is stated per table. Every 422/409/503 row also asserts: no appointment created, balance unchanged, no new ledger row, no new outbox row. |

## B1. Boundary tables

### AC-2 Minimum redemption (≥100)
Isolation: fee ₹10,000 (cap 5000, so the max never binds), balance 10,000, values on the 10-step (so the step rule never binds).

| Col | ID | points_to_redeem | Expected |
|---|---|---|---|
| below-min | CP-2-1 | 90 | 422 `points_below_minimum` |
| min | CP-2-2 | 100 | 201; `points_redeemed`=100, `net_fee`=9900; balance 9900 |
| min+step | CP-2-3 | 110 | 201; `net_fee`=9890 |
| max-step | — | n/a | One-sided rule; the upper side is AC-4 (§1 AC-4 table) |
| max | — | n/a | as above |
| above-max | — | n/a | as above |
| extra | CP-2-4 | 10 (smallest on-step value) | 422 `points_below_minimum` |
| extra | CP-2-5 | 0 and omitted | 201; no redemption, `points_redeemed`=0, `net_fee`=10000, no ledger row (AC-1) |
| precedence | CP-2-6 | 1, 99 (off-step **and** below min) | 422 `points_below_minimum` (AC-2 explicitly names 1–99), not `invalid_points_step` |

### AC-3 Redemption step (multiples of 10; integers ≥0)
Isolation: fee ₹10,000, balance 10,000; all values are within [100, 5000], so the min and max never bind.

| Col | ID | points_to_redeem | Expected |
|---|---|---|---|
| below-min | CP-3-1 | 99 | covered by CP-2-6 (min takes precedence) |
| min | CP-3-2 | 100 | 201 |
| min+1 (off-step) | CP-3-3 | 101 | 422 `invalid_points_step` |
| min+step−1 | CP-3-4 | 109 | 422 `invalid_points_step` |
| min+step | CP-3-5 | 110 | 201 |
| mid off-step | CP-3-6 | 105, 2345 | 422 `invalid_points_step` |
| max-step | CP-3-7 | 4990 | 201 |
| max−1 (off-step) | CP-3-8 | 4999 | 422 `invalid_points_step` |
| max | CP-3-9 | 5000 | 201 |
| above-max | — | 5010 | covered by AC-4 |
| type: negative | CP-3-10 | −10, −100 | 422 (`validation_error`) |
| type: non-integer | CP-3-11 | 100.5, 110.0 | 422 for 100.5. **Open Q1:** is 110.0 accepted? (Pydantic lax mode accepts it; strict mode rejects it.) |
| type: wrong JSON type | CP-3-12 | "100", true, [], {} | 422. **Open Q2:** is "100" coerced? (Same strict/lax question.) |
| type: null | CP-3-13 | null | **Open Q3:** treated as omitted (201) or 422? |

### AC-4 Maximum redemption (≤ floor(50% × fee) rounded down to 10)
Isolation: balance 10,000; values ≥100 and multiples of 10, so only the cap can fail.

| Fee | Cap | max-step | max | above-max | IDs |
|---|---|---|---|---|---|
| ₹10,000 | 5000 | 4990 → 201 | 5000 → 201, net 5000 | 5010 → 422 `exceeds_max_redemption` | CP-4-1..3 |
| ₹500 | 250 | 240 → 201 | 250 → 201, net 250 | 260 → 422 | CP-4-4..6 |
| ₹510 (rounding) | 250 | 240 → 201 | 250 → 201, net 260 | 260 → 422 (50% = 255 is on neither step) | CP-4-7..9 |
| ₹200 (cap = min) | 100 | n/a (90 is below min) | 100 → 201, net 100 | 110 → 422 | CP-4-10..11 |
| ₹190 (cap < min) | 90 | — | — | see §3 | — |

Extra:
- CP-4-12: fee ₹510, points 255 (= raw 50%, off-step) → 422 `invalid_points_step`. This proves rounding is not done by rejecting; 255 fails on the step rule.
- CP-4-13: fee ₹10,000, points 10,000,000 (very large) → 422 `exceeds_max_redemption`, no 500 or overflow.
- Min side: covered by the AC-2 table.

### AC-8 Cancellation refund (≥24h before `starts_at` refunds)
Isolation: fee ₹10,000, redeem 100 (smallest valid), balance 10,000 → 9,900 after booking. Book at `FIXED_NOW` with `starts_at` = S (e.g. 2030-03-04 09:00Z). Then set the clock to `S − Δ` and cancel (BOOKED, correct version).

| Col | ID | Δ = starts_at − cancel time | Expected after cancel |
|---|---|---|---|
| below-min | CP-8-1 | 23h59m59s | 200 CANCELLED; forfeit: balance 9,900; no refund row; no zero-value row |
| below-min (sub-second) | CP-8-2 | 23h59m59.999999s | forfeit (refund only at ≥ 24h exactly) |
| min | CP-8-3 | 24h00m00s exactly | 200; refund: balance 10,000; one +100 ledger row |
| min+step | CP-8-4 | 24h00m01s | refund |
| max-step / max | CP-8-5 | Δ = `S − FIXED_NOW` (cancel at the booking instant, largest possible Δ) | refund |
| above-max | — | n/a | Rule is unbounded above |
| at start | CP-8-6 | 0s (cancel at `starts_at`) | forfeit |
| after start | CP-8-7 | −1s (BOOKED, start passed) | forfeit (current service still allows cancelling BOOKED) |
| no points | CP-8-8 | 25h and 1h, `points_redeemed`=0 | 200; **no** ledger row in either case (no zero-value entries) |

### AC-10 Earning on COMPLETED (5% of `net_fee`, rounded down)
Isolation: book → check-in → complete. Assert the ledger earn row and the balance delta. Seed balance where a redemption is needed.

| Col | ID | Fee / redeemed → net | Earn |
|---|---|---|---|
| below-min | CP-10-1 | ₹0 / 0 → 0 | 0, **no** ledger row |
| min | — | net ₹20 (first value earning 1 point) | **Gap G-A:** net ₹19/₹20 can't be reached with the given schedule (smallest non-zero net is ₹100). Proposal: add ₹19 and ₹20 to the test fee config, or accept this as untested. |
| min+step | CP-10-2 | ₹200 / 100 → 100 | 5 |
| rounding | CP-10-3 | ₹190 / 0 → 190 | 9 (9.5 → 9) |
| rounding | CP-10-4 | ₹500 / 250 → 250 | 12 (12.5 → 12) |
| rounding | CP-10-5 | ₹510 / 0 → 510 | 25 (25.5 → 25) |
| rounding | CP-10-6 | ₹510 / 250 → 260 | 13 |
| exact | CP-10-7 | ₹200 / 0 → 200 | 10 |
| max-step | CP-10-8 | ₹10,000 / 5000 → 5000 | 250 |
| max | CP-10-9 | ₹10,000 / 0 → 10,000 | 500 |
| above-max | — | n/a | Rule is unbounded; the largest schedule value is used |
| no-show | CP-10-10 | ₹10,000 / 100, then no-show | Earns 0; redeemed 100 forfeited (balance stays −100 from the seed); no refund row |
| not on booking | CP-10-11 | any booking, check-in only | Earns 0 at book and at check-in |
| cancel | CP-10-12 | completed? n/a; cancel ≥24h | Earns 0 (only the refund per AC-8) |
| once only | CP-10-13 | complete, then retry complete (409) | Exactly one earn row |

### AC-14 Expiry (365 days) and consumption order
Isolation: fee ₹10,000, redeem 100. Seed **one** lot of 100 points `earned_at = E` (E = 2030-01-15 08:00Z), then move the clock to `E + n days` and book.
**Open Q4:** "day N" is ambiguous: is it the instant E + N×24h, or UTC calendar days? Rows CP-14-4a/4b distinguish the two.

| Col | ID | Booking time | Expected |
|---|---|---|---|
| min | CP-14-1 | E (same instant) | 201; lot consumed |
| min+step | CP-14-2 | E + 1 day | 201 |
| max-step | CP-14-3 | E + 364 days | 201 |
| max | CP-14-4a | E + 365 days exactly | 201 (day 365 is still valid) |
| max (calendar) | CP-14-4b | E + 365 days + 15h59m59s (23:59:59Z on day 365) | 201 under the calendar reading; 409 under the instant reading. Pin after Q4 is answered. |
| above-max | CP-14-5 | E + 366 days | 409 `insufficient_points`; balance shows 0 |
| leap year | CP-14-6 | Lot earned 2031-03-01; book on 2032-02-29 (day 365, leap day) and on 2032-03-01 (day 366) | 201 / 409 |
| balance endpoint | CP-14-7 | `GET /patients/{id}/points` at day 365 and day 366 | Available 100 / 0 (expired points excluded) |

Consumption order (FIFO over unexpired lots):

| ID | Lots (earned) | Action | Expected |
|---|---|---|---|
| CP-14-8 | A = 100 (day −300), B = 200 (day −10) | Redeem 150 | A fully consumed, B has 150 left; ledger debit references A then B |
| CP-14-9 | Same; then move the clock to day +66 (A's expiry) | GET balance | 150 (nothing left of A to expire) |
| CP-14-10 | A = 100 (day −300), B = 200 (day −10); redeem 100 | Then advance to A's day 366 | Balance 200 (A was consumed first; nothing lost) |
| CP-14-11 | A = 100 (day −400, expired), B = 100 (day −10) | Redeem 100 | 201 from B; A is not counted |
| CP-14-12 | A = 100 expired only | Redeem 100 | 409 `insufficient_points` |
| CP-14-13 | A, B; redeem 150, then cancel ≥24h | Refund | **Open Q5:** do refunded points go back to their original lots (keeping their expiry) or into a new lot? |

## B2. Decision table — cancellation refund (AC-8, AC-9)
Conditions: **T** = time before `starts_at` when cancelling (≥24h / <24h); **S** = status; **P** = points redeemed (>0 / 0). Base: fee ₹10,000, P=100, balance 9,900 after booking.

| Rule | T | S | P | Cancel response | Ledger | Balance after | ID |
|---|---|---|---|---|---|---|---|
| R1 | ≥24h | BOOKED | >0 | 200 CANCELLED | +100 refund row | 10,000 | CP-DT-1 |
| R2 | <24h | BOOKED | >0 | 200 CANCELLED | none | 9,900 | CP-DT-2 |
| R3 | ≥24h | BOOKED | 0 | 200 | none (no zero row) | unchanged | CP-DT-3 |
| R4 | <24h | BOOKED | 0 | 200 | none | unchanged | CP-DT-4 |
| R5 | any | CHECKED_IN | >0 | 409 `conflict` | none | 9,900 | CP-DT-5 |
| R6 | any | COMPLETED | >0 | 409 | none (earn row from completion only) | 9,900 + earn | CP-DT-6 |
| R7 | any | NO_SHOW | >0 | 409 | none | 9,900 | CP-DT-7 |
| R8 | ≥24h | CANCELLED (2nd cancel after R1) | >0 | 409 | **no second refund** | 10,000 | CP-DT-8 |
| R9 | – (both values) | CHECKED_IN/COMPLETED/NO_SHOW/CANCELLED | 0 | 409 | none | unchanged | CP-DT-9 (parametrised) |

AC-9 reschedule rows (book with S1, reschedule to S2 with the correct version, then cancel):

| ID | S1 vs cancel time | S2 vs cancel time | Expected |
|---|---|---|---|
| CP-DT-10 | ≥24h | <24h (moved earlier) | forfeit (the new `starts_at` decides) |
| CP-DT-11 | <24h | ≥24h (moved later) | refund 100 |
| CP-DT-12 | — | exactly 24h00m00s from the new start | refund |
| CP-DT-13 | reschedule only | — | `consultation_fee`, `points_redeemed`, `net_fee` unchanged; no ledger row; balance unchanged |

## B3. Interaction cases: 50% cap below the 100-point minimum
Arises for fees < ₹200 (schedule: ₹190, cap 90). Isolation: balance 10,000.

| ID | Fee | points_to_redeem | Rules violated | Expected |
|---|---|---|---|---|
| CP-INT-1 | ₹190 | 100 | max only | 422 `exceeds_max_redemption` |
| CP-INT-2 | ₹190 | 90 (= cap) | min only | 422 `points_below_minimum` |
| CP-INT-3 | ₹190 | 10 | min only | 422 `points_below_minimum` |
| CP-INT-4 | ₹190 | 95 | min + step | 422 `points_below_minimum` (AC-2 precedence, as CP-2-6) |
| CP-INT-5 | ₹190 | 110 | max only | 422 `exceeds_max_redemption` |
| CP-INT-6 | ₹190 | 0 / omitted | none | 201; fee 190, net 190 (patient can still book) |
| CP-INT-7 | ₹190 | any value 1..∞ | no value is valid | Parametrised sweep 10..200 step 10 → all 422; no 201 |
| CP-INT-8 | ₹200 | 100 | none (cap = min) | 201; net 100. The only valid value for this fee. |
| CP-INT-9 | ₹200 | 90 / 110 | min / max | 422 `points_below_minimum` / `exceeds_max_redemption` |
| CP-INT-10 | ₹0 | 100 | fee-zero (+ max, cap 0) | 422 `redemption_not_allowed` (AC-6 takes precedence over AC-4) |
| CP-INT-11 | ₹0 | 10, 95 | fee-zero + min + step | 422 `redemption_not_allowed` |
| CP-INT-12 | ₹0 | 0 / omitted | none | 201; net 0 |
| CP-INT-13 | ₹190 | 100, balance 0 | max (422) + balance (409) | 422 `exceeds_max_redemption` (validation before balance; **Open Q6**) |

**Proposed precedence (Open Q6, to confirm):**
1. 401
2. 403
3. 503 (`APPOINTMENTS_ENABLED`, then `POINTS_ENABLED`)
4. 422 schema (type, negative, extra field)
5. `redemption_not_allowed`
6. `points_below_minimum`
7. `invalid_points_step`
8. `exceeds_max_redemption`
9. 409 overlap `conflict`
10. 409 `insufficient_points`

The spec states only AC-2 > AC-3 (via "1–99") and implies AC-6 first.

## B4. Negative paths

### 4.1 Invalid input (fee ₹10,000, balance 10,000)
| ID | Input | Expected |
|---|---|---|
| CP-NEG-1 | `points_to_redeem` on the reschedule body | 422 (`extra_forbidden`; redemption only at booking, AC-1) |
| CP-NEG-2 | `points_to_redeem` on the cancel or check-in body | 422 |
| CP-NEG-3 | Client-sent `consultation_fee`, `net_fee`, `points_redeemed` or `balance` on booking | 422 (server computes them; never trusted) |
| CP-NEG-4 | Malformed patient id on `GET /patients/{id}/points` | 422 |
| CP-NEG-5 | 422 body for any points error | Does not echo the submitted value, fee or balance (existing sanitiser + AC-15) |

### 4.2 Insufficient balance (AC-5); fee ₹10,000, points 500
| ID | Balance | Expected |
|---|---|---|
| CP-NEG-6 | 490 | 409 `insufficient_points`; no appointment; balance 490 |
| CP-NEG-7 | 500 (exact) | 201; balance 0 |
| CP-NEG-8 | No ledger at all | 409; `GET` points → balance 0 (200, not 404) |
| CP-NEG-9 | 500, all expired | 409 (see CP-14-12) |
| CP-NEG-10 | 499 points across 2 lots | 409; neither lot touched |

### 4.3 Flags off (AC-16, AC-7)
| ID | Flags | Request | Expected |
|---|---|---|---|
| CP-NEG-11 | default settings | `points_enabled` default | `False` (config test); `.env.example` has `POINTS_ENABLED=false` |
| CP-NEG-12 | POINTS off | Book with points 100 | 503 `feature_disabled`; no appointment, no debit |
| CP-NEG-13 | POINTS off | Book with points 0 / omitted | 201; response fee fields per Open Q7 |
| CP-NEG-14 | POINTS off | Book with points 99 (invalid) | 503 (flag before validation, matching current appointment-flag behaviour; confirm with Q6) |
| CP-NEG-15 | APPOINTMENTS off, POINTS on | Book with points 100 | 503; no debit (AC-7) |
| CP-NEG-16 | POINTS off after a redemption | Cancel ≥24h | **Open Q8:** refund still issued? Proposed: yes (ledger integrity), but the spec is silent |
| CP-NEG-17 | POINTS off | Complete | **Open Q8:** earn or not? |
| CP-NEG-18 | POINTS off | `GET /patients/{id}/points` | **Open Q8** |
| CP-NEG-19 | POINTS off | Unauthenticated / DOCTOR booking with points | 401 / 403 (auth before flag) |

### 4.4 Conflicts and atomicity (AC-7, AC-11, AC-13)
| ID | Env | Scenario | Expected |
|---|---|---|---|
| CP-NEG-20 | S | Overlapping slot, points 100 | 409 `conflict`; balance unchanged; no ledger row |
| CP-NEG-21 | S | `add_event` patched to raise (existing outbox failure helper), points 100 | 500; no appointment; no debit |
| CP-NEG-22 | S | Ledger write patched to raise | 500; no appointment; no outbox row |
| CP-NEG-23 | P | 2 threads, different slots, balance 100, each redeems 100 (barrier after balance read) | `[201, 409 insufficient_points]`; balance 0; never negative |
| CP-NEG-24 | P | 10 threads, balance 500, each redeems 100, different slots | Exactly 5 × 201; balance 0; ledger debits sum to 500 |
| CP-NEG-25 | P | 2 threads, **same** slot, both with points 100, balance 200 | `[201, 409 conflict]`; exactly one debit; balance 100 |
| CP-NEG-26 | P | Concurrent cancel (≥24h) ×2, same version | `[200, 409]`; exactly one refund row |
| CP-NEG-27 | S | Book same slot without points, then with points | Same 409 as without points (AC-11) |
| CP-NEG-28 | S | Scheduling advice says unavailable, points 100 | 409; no debit; fake called exactly as without points |

### 4.5 Roles and minimum data (AC-12)
| ID | Role / token | Request | Expected |
|---|---|---|---|
| CP-NEG-29 | ADMIN, RECEPTIONIST | `GET /api/v1/patients/{id}/points` | 200 |
| CP-NEG-30 | DOCTOR, READ_ONLY | same | 403 `forbidden` |
| CP-NEG-31 | none / invalid / expired | same | 401 + `WWW-Authenticate` |
| CP-NEG-32 | DOCTOR | GET appointment, search, check-in/complete/no-show responses | No `consultation_fee`, `points_redeemed` or `net_fee` keys (exact key set = the current 13 fields) |
| CP-NEG-33 | ADMIN, RECEPTIONIST | Same responses | Fee and points fields present |
| CP-NEG-34 | Multi-role `[DOCTOR, RECEPTIONIST]` | GET appointment | **Open Q9:** are fee fields shown? (Proposed: shown, because the RECEPTIONIST permission is present) |
| CP-NEG-35 | DOCTOR | Book with points | 403 (cannot book at all) |
| CP-NEG-36 | Reuse | Add the points route to the `ENDPOINTS` matrix with `{ADMIN, RECEPTIONIST}` | 401/403 matrix extends automatically |

---

## C. Test conventions (apply to every case)

- **C1 Seeded patient:** every points case books for the seeded patient (`points_patient` fixture → `book(provider, patient_id=points_patient)`). `booking_payload()` alone uses a random patient with balance 0, which would fail 409 for the wrong reason.
- **C2 Effects, not just statuses:**
  - Every success asserts the exact debit, refund or earn ledger row (amount and count) plus the resulting balance.
  - Every rejection compares appointment count, balance, ledger rows and outbox rows before and after.
- **C3 Clock:**
  - All times come from `app.state.clock`.
  - When the clock moves, `starts_at = clock + 7 days`.
  - Seeded lots set `earned_at` explicitly.
  - No `sleep`, and no `datetime.now()` except for token minting.
- **C4 Concurrency (PostgreSQL only):**
  - The `threading.Barrier` sits before ledger access, never after a locked read.
  - Each thread gets a distinct provider and slot, unless overlap is the target.
  - Each thread gets a distinct patient, unless balance contention is the target.
  - Every race also runs 5 times without the barrier; barrier timeout is 10 s.
- **C5 Privacy checks** use distinctive amounts (e.g. 86,420; 4321; 9876), never common numbers.
- **C6 "Complete"** means check-in and then complete, each with the current `version`.

## D. Open questions, assumptions and gaps

Expected results use the defaults below until the questions are answered. An answer may change an expected result; it doesn't remove the case.

| # | Question | Default used |
|---|---|---|
| Q1/Q2 | Strict integer typing for `points_to_redeem` (reject `110.0` and `"100"`)? | Strict: both rejected with 422 |
| Q3 | `points_to_redeem: null` is the same as omitted? | Yes |
| Q4 | Is expiry day 365 an instant (`earned_at + 365 × 24 h`) or a UTC calendar day? | Instant: valid while `now ≤ earned_at + 365 d` |
| Q5 | Do refunded points return to their original lots (and expiry)? | Yes |
| Q6 | Order of error checks across rules | 401 → 403 → 503 → 422 schema → `redemption_not_allowed` → `points_below_minimum` → `invalid_points_step` → `exceeds_max_redemption` → 409 overlap → 409 `insufficient_points` |
| Q7 | With `POINTS_ENABLED` off, are fee fields still shown? | Yes |
| Q8 | With `POINTS_ENABLED` off, do refunds, earning and the balance endpoint still work? | Yes |
| Q9 | Fee visibility for a multi-role token that includes DOCTOR | Shown if any role grants it |
| Q10 | Must two concurrent redemptions that both fit in the balance both succeed? | Not required; a retryable 409 is acceptable, and overspending never is |
| A1 | How a booking gets its fee | Test-config mapping (e.g. by `reason_code`) |
| G-A | The schedule can't reach net ₹19/₹20, the first earning threshold | Add ₹19 and ₹20 to the test fee config (CP-10-14) |
