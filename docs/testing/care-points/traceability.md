# Care Points — Traceability

Maps each Care Points acceptance criterion (AC-1..AC-17) to the case IDs in [`test-cases.csv`](test-cases.csv). The test plan is in [`test-plan.md`](test-plan.md).

**Status:** design only. The feature is not implemented yet, and no tests have been written.

A case that covers several ACs appears in every matching row.

| AC | Rule | Positive | Boundary | Negative | Regression | Total |
|---|---|---|---|---|---|---|
| AC-1 | Redeem at booking | CP-1-1, CP-1-2, CP-1-7, CP-INT-6 | CP-1-3 | CP-1-4, CP-1-5, CP-1-6 | — | 8 |
| AC-2 | Minimum redemption | CP-3-P1 | CP-1-3, CP-2-1, CP-2-2, CP-2-3, CP-2-U1 | CP-2-4, CP-2-6, CP-INT-2, CP-INT-3, CP-INT-4 | — | 11 |
| AC-3 | Redemption step | CP-3-P1 | CP-2-2, CP-2-3, CP-3-3, CP-3-4, CP-3-7, CP-3-8, CP-3-9, CP-3-13, CP-3-U1 | CP-2-6, CP-3-6, CP-3-10, CP-3-11, CP-3-12 | — | 15 |
| AC-4 | Maximum redemption | CP-4-P1 | CP-3-7, CP-3-9, CP-4-3, CP-4-4, CP-4-5, CP-4-6, CP-4-7, CP-4-8, CP-4-9, CP-4-10, CP-4-11, CP-4-U1 | CP-4-12, CP-4-13, CP-INT-1, CP-INT-5, CP-INT-7, CP-INT-9 | — | 19 |
| AC-5 | Available balance | CP-5-4 | CP-5-2 | CP-5-1, CP-5-3, CP-5-5, CP-5-6, CP-INT-13 | — | 7 |
| AC-6 | Zero-fee appointments | CP-6-3 | — | CP-6-1, CP-6-2 | — | 3 |
| AC-7 | Atomic with booking | CP-7-P1 | — | CP-7-1, CP-7-2, CP-7-3, CP-7-4, CP-7-5 | — | 6 |
| AC-8 | Cancellation refund | CP-8-5 | CP-8-1, CP-8-2, CP-8-3, CP-8-4, CP-8-6, CP-8-8, CP-8-U1 | CP-8-7, CP-DT-5, CP-DT-6, CP-DT-7, CP-DT-8, CP-DT-9 | — | 14 |
| AC-9 | Reschedule | CP-DT-13 | CP-DT-10, CP-DT-11, CP-DT-12 | CP-9-1 | — | 5 |
| AC-10 | No-show, completion and earning | CP-10-P1 | CP-8-3, CP-10-1, CP-10-2, CP-10-3, CP-10-4, CP-10-5, CP-10-6, CP-10-7, CP-10-8, CP-10-9, CP-10-14, CP-10-U1 | CP-10-10, CP-10-11, CP-10-13 | — | 16 |
| AC-11 | No effect on the clinical appointment | CP-11-P1 | — | CP-NEG-25, CP-11-N1, CP-11-N2 | CP-11-1, CP-11-2, CP-11-3, CP-11-4, CP-11-5, CP-11-6, CP-11-9 | 11 |
| AC-12 | Roles and minimum data | CP-NEG-29, CP-NEG-33 | CP-NEG-34 | CP-NEG-4, CP-NEG-30, CP-NEG-31, CP-NEG-32, CP-NEG-36 | — | 8 |
| AC-13 | Concurrent redemptions | CP-13-P1 | — | CP-NEG-23, CP-NEG-24, CP-NEG-25, CP-NEG-26, CP-NEG-26b | — | 6 |
| AC-14 | Expiry and consumption order | CP-14-7, CP-14-8, CP-14-13 | CP-14-1, CP-14-2, CP-14-3, CP-14-4a, CP-14-4b, CP-14-5, CP-14-6, CP-14-9, CP-14-U1 | CP-5-5, CP-14-11 | — | 14 |
| AC-15 | Data minimisation | CP-15-P1 | — | CP-NEG-5, CP-15-1, CP-15-2, CP-15-3 | — | 5 |
| AC-16 | Feature flag | CP-NEG-11, CP-NEG-13, CP-NEG-16, CP-NEG-17, CP-NEG-18 | — | CP-7-4, CP-NEG-11b, CP-NEG-12, CP-NEG-14, CP-NEG-19 | — | 10 |
| AC-17 | Schema change safety | CP-17-1, CP-17-2 | — | CP-17-3, CP-17-4 | — | 4 |

**Totals:** 151 cases.
- **By type:** positive 27, boundary 54, negative 63, regression 7.
- **By level:** API 133, Unit 8, PostgreSQL 6, Migration 4. PostgreSQL-level cases need `TEST_DATABASE_URL`.
- **By existing asset:** EXTEND 49, NEW 102.

## Coverage checks

- ACs without a positive case: none.
- ACs without a boundary or negative case: none.
- Cases without an AC: none.

**Self-review.** Every case was checked against one question: "if the feature were broken the way this case targets, would it fail?"
- **Findings:** 68 — Tautology / internal duplicate 20, Duplicate of existing test 2, Status-only 19, Unclear oracle 4, Assertion on mock 1, Can fail for a different reason 20, Traceability 1, Clock 1.
- **Result:** all were fixed. Revised rows are marked `revised_in_self_review=yes` in the CSV.
- **Removed:** duplicates and alias rows. Each one's AC is now traced by the case that remains.

## Dependencies and open items

- **Open questions Q1–Q10.** Expected results use the stated defaults (see the test plan, section 9); if an answer changes a default, the expected result changes.
  - CP-3-11/12/13: Q1–Q3
  - CP-14-4b: Q4
  - CP-14-13: Q5
  - CP-INT-13 and CP-4-12: Q6
  - CP-NEG-13: Q7
  - CP-NEG-16/17/18: Q8
  - CP-NEG-34: Q9
  - CP-13-P1: Q10
- **CP-10-14** needs ₹19 and ₹20 added to the test fee config (gap G-A).
- **Fee lookup (A1):** every fee-dependent case assumes a test-config mapping from booking to fee.
- **Planned change to an existing test:** CP-NEG-32 changes the DOCTOR equality assertion in `services/appointment-service/tests/test_appointments_api.py::test_get_appointment` (AC-12).
- **AC-13** is proven only on PostgreSQL. SQLite results are not evidence for concurrency.
