# Devin playbook v2 — Smart Hospital: Analyze, Plan & Build

Paste the text below into Devin -> Settings -> Connections -> Jira -> Playbooks, replacing the body of
"Smart Hospital — Analyze, Plan & Build". Keep the macro `!smart_hospital__analyze_plan__build`.
Changes from v1 are marked **(v2)**.

---

You are working on the Smart Hospital & Patient Management System, a Python FastAPI microservices
project. Follow `docs/workflow/governed-sdlc.md` in the repository; this playbook is how you carry out
checkpoints CP1 and CP2 of that workflow.

WORKFLOW

1. Read the Jira work item. Identify the objective, scope, constraints and acceptance criteria.
2. Read README.md, AGENTS.md, Smart_Hospital_Repository_Rules.md and docs/workflow/governed-sdlc.md.
3. Inspect the repository: the closest existing service, its structure, config, token contract and tests.
4. **(v2) CP1 — Plan gate.** Post a plan as a comment on the Jira issue using the Atlassian/Jira tools:
   assumptions, files to add, API with roles per endpoint, data model and migration, dependencies,
   security implications, test plan, and — for patient-facing behaviour — the feature flag and rollback
   approach. Then STOP and wait. Do not create or edit files until a human replies "Plan approved"
   (in the session or on the Jira issue). If changes are requested, revise the plan and wait again.
5. Implement only what the approved plan and the Jira item cover.
6. **(v2)** Put new patient-facing behaviour behind a configuration feature flag that defaults to OFF.
   Keep migrations additive (expand, then contract) and give every migration a working downgrade.
7. Add or update tests: success, 422, 404, 409, 401 and 403 cases.
8. **(v2) CP2 — Self-check.** Run and record: `ruff check .`, `ruff format --check .`, `pytest`,
   `pytest` with `TEST_DATABASE_URL` when PostgreSQL is available, `alembic upgrade head`,
   `alembic downgrade base`, `alembic upgrade head`, `alembic check`. Fix failures caused by your change.
9. Review the diff for unrelated changes, secrets, unsafe logs and scope violations.
10. **(v2)** Open a PR against main using `.github/pull_request_template.md`; fill every section,
    including checkpoint evidence and the rollback plan. Do not merge.
11. **(v2)** Post the PR link and the completion report as a comment on the Jira issue.
12. If CI fails on the PR (CP3) or a reviewer requests changes (CP4), fix it on the same branch.

GUARDRAILS

- Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2.x, Alembic, PostgreSQL, pytest.
- Do not invent repository structure; inspect it first.
- Do not implement unrelated microservices or features; do not modify other services.
- Synthetic data only. Never hard-code secrets. Never log tokens, patient contact details, dates of birth
  or note contents.
- No Docker, Kubernetes, frontend, cloud deployment or new CI/CD jobs unless the Jira item asks.
- Never merge, force-push main, or drop/delete persistent data.
- If a critical requirement is unclear, ask one focused question instead of guessing.
- The Jira acceptance criteria are the completion checklist.
