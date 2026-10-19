## Summary

<!-- What this PR changes, in two or three sentences. -->

- Jira issue: KAN-
- Service(s): `services/...`
- Workflow version: `docs/workflow/governed-sdlc.md` v1.0

## API / schema / migration changes

<!-- Endpoints added or changed (method, path, roles). Tables/columns added. Alembic revision IDs. -->

## Checkpoint evidence

| # | Checkpoint | Evidence |
|---|---|---|
| CP0 | Definition of Ready | <!-- link to the ticket in Ready for Devin --> |
| CP1 | Plan approved | <!-- link to the plan comment and the approval --> |
| CP2 | Agent self-check | <!-- commands run below, with PASS/FAIL --> |
| CP3 | CI gate | Green `CI` checks on this PR |
| CP4 | Human review | Reviewer checklist below |

### Commands run (CP2)

| Command | Result |
|---|---|
| `ruff check .` | PASS / FAIL |
| `ruff format --check .` | PASS / FAIL |
| `pytest` (SQLite) | PASS / FAIL — N tests |
| `pytest` with `TEST_DATABASE_URL` (PostgreSQL) | PASS / FAIL / NOT RUN |
| `alembic upgrade head` -> `downgrade base` -> `upgrade head` | PASS / FAIL / NOT RUN |
| `alembic check` | PASS / FAIL / NOT RUN |

## Rollback plan

Patient-facing change? **Yes / No**

<!-- Required when Yes. -->

- Feature flag: `NAME` (default OFF) — how to switch it off:
- Code revert: `git revert -m 1 <merge-commit>` via a new PR
- Migration: additive only? Yes / No. `downgrade` tested? Yes / No. Data-loss risk on downgrade:
- Rollback triggers agreed for this change:

## Security and boundary review

- [ ] No `.env`, secrets, tokens or real data committed; `.env.example` has placeholders only
- [ ] Only files under the target service (and root README / docs) changed
- [ ] No access to another service's database or tables
- [ ] Every protected endpoint enforces roles server-side, with 401/403 tests
- [ ] No passwords, tokens, patient contact details, dates of birth or note contents in logs
- [ ] No diagnosis, treatment or clinical decision logic

## Reviewer checklist (CP4)

- [ ] Acceptance criteria in the Jira issue are all met
- [ ] Tests cover success, validation (422), not found (404), conflict (409), 401 and 403
- [ ] Migration matches the models and follows expand-then-contract
- [ ] Rollback plan is present and credible (or the change is not patient-facing)
- [ ] CI is green and the reported results match the CI run

## Known limitations

<!-- What is not done, assumptions made, follow-ups. -->
