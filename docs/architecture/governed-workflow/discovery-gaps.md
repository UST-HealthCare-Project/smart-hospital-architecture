# Discovery Gaps and Unknowns (governed workflow — evidence pack, part 4/4)

What was **not** inspected, what is **unknown**, and where evidence is **only** documentation or mocks.

## 1. Uninspected areas

| # | Area | Why it matters | Status |
|---|---|---|---|
| U-1 | Jira (KAN / UXBA) and qTest project 32916 requirements and test cases | They may hold notification or non-functional requirements | Not inspected |
| U-2 | Datadog live state (whether the SLOs, monitors and dashboards exist and what they show) | §H relies on Terraform definitions; the dashboards were created via the API | Not inspected |
| U-3 | Argo CD sync history, Kubernetes events, pod logs | Operational behaviour over time | Only point-in-time status was checked (E-23) |
| U-4 | AWS account-level controls (CloudTrail, GuardDuty, SCPs, budgets) | Security and cost posture | Not inspected |
| U-5 | Frontend / API gateway / prototypes (`prototypes/booking-ui/` mentioned earlier is not on `main`) | Consumer side of every API | No repository exists |
| U-6 | Figma design kit | UX for notifications | Not inspected |
| U-7 | Commits on the monorepo before the split (`git log` history) | Rationale for past decisions | Not inspected beyond file contents |

## 2. Unknowns (no evidence either way)

| # | Unknown | Related step |
|---|---|---|
| K-1 | Real traffic volumes, latencies and error rates | G, H |
| K-2 | Who calls the APIs beyond the smoke test (external consumers) | C |
| K-3 | Whether the Datadog SLO targets (99.5 %, 500 ms) were agreed by the business or are module defaults | H |
| K-4 | RPO/RTO expectations; whether RDS restores have been rehearsed | D, G |
| K-5 | Patient consent and contact-preference ownership | D, F |
| K-6 | Support/on-call model and incident runbooks | H |
| K-7 | Whether `clinical-platform` or `dev` is the intended long-term platform | E |
| K-8 | When the org repos replace the monorepo as the delivery source | A, E |

## 3. Evidence that is documentation-only, mock-only or stale

| # | Item | Limitation |
|---|---|---|
| L-1 | SLO and dashboard existence in Datadog | DOC/CONFIG only (E-47, U-2) |
| L-2 | Cross-service client behaviour | Verified with **mocks** (E-40, E-41); no provider-side contract tests (E-42) |
| L-3 | Infrastructure tests | Mocked providers (E-53); the real apply has worked twice (LIVE) |
| L-4 | Rollback procedures | DOC (E-49); not rehearsed in this inspection |
| L-5 | R7 `deploy/helm/db-bootstrap/dev-values.yaml` | **Stale** against the monorepo (A, drift check) |
| L-6 | Org repos have no CI on `main` | CI exists only on an unmerged branch (A) |
| L-7 | Architecture pack documents (discovery, dependency map, trade-off, ADR-001, ARB review, decision brief) | On unmerged branches; `main` of R9 doesn't contain them |
| L-8 | The ALB being HTTP-only (E-35) | Inferred from the ACM/domain logic; listener protocol not read line by line |

## 4. Next inspection actions (no technology implied)

1. Read Jira and qTest for requirements (U-1).
2. Read-only Datadog query of the SLO, monitor and dashboard inventory (U-2).
3. Ask the owners for K-1 to K-8 (they map to decision-brief questions Q2–Q12).
4. Re-sync R7 with the monorepo, or record the monorepo as the authoritative deployment source (L-5).
5. Merge or close the architecture-pack branches so evidence has a stable base (L-7).
