# Governed SDLC Workflow — v1.0

How a Jira story becomes reviewed, mergeable code in this repository when the code is written by Devin.
The workflow was derived from the Patient Service story (KAN-6) and is validated on a second, different
story before it is called reusable.

> Development demo only. Synthetic data only. No diagnosis, treatment or clinical decision logic.
> Passing these checkpoints does not make the system clinically or regulatorily compliant.

## 1. Flow

```
Jira story --CP0--> Devin plans --CP1--> Devin builds --CP2--> Pull request --CP3/CP4--> main --CP5--> live
                                                                     ^                                  |
                                                                     +---- revert re-enters as a PR ----+
```

A failed checkpoint sends the work back one stage, never forward. Rollback never bypasses a checkpoint:
a revert is a new pull request and passes CP3 and CP4 again.

## 2. Checkpoints

| # | Checkpoint | Placed at | Verifies | Evidence | Owner | Type |
|---|---|---|---|---|---|---|
| CP0 | Definition of Ready | Jira: To Do -> Ready for Devin | All 8 story sections filled (Objective, Parts 1-5, Scope, Acceptance Criteria); acceptance criteria are testable; "do not touch" list present; synthetic data only | Ticket status change + DoR checklist on the ticket | Tech lead / product owner | Human |
| CP1 | Plan approval | Devin session, before any code | Plan respects the service boundary and own database, data model, roles per endpoint, test plan, rollback approach for patient-facing behaviour, nothing out of scope | Plan comment on the Jira issue + "Plan approved" reply | Tech lead | Human |
| CP2 | Agent self-check | Devin session, before the PR | Tests written and passing, lint and format clean, `alembic upgrade head` / `downgrade base` / `upgrade head` run, diff reviewed for secrets and scope | Completion report with commands and PASS/FAIL | Devin | Automated (agent) |
| CP3 | CI gate | GitHub, every PR push | Independent clean-machine re-run: ruff, pytest on SQLite and PostgreSQL, Alembic round trip + `alembic check`, secret scan | Green `CI` checks on the PR | GitHub Actions (`.github/workflows/ci.yml`) | Automated |
| CP4 | Human code review | GitHub PR | Correctness; auth on every protected endpoint; no patient data, tokens or contact details in logs; migration safety; rollback section present and credible | Approved review + completed checklist in the PR template | Reviewer who is not the ticket author | Human |
| CP5 | Release gate | Before a patient-facing change is switched on | Smoke test passes; feature flag defaults OFF; rollback rehearsed; owner available | Smoke result + release approval comment on the ticket | Release owner | Human + automated |

Until a staging environment exists (future scope), CP5 is a local smoke test plus a rehearsed rollback.

## 3. Definition of Ready (CP0)

A ticket may move to **Ready for Devin** only when:

- [ ] Summary follows `<Service name> - Endpoints, Tests and Pull-Request Readiness` (or a clear change title)
- [ ] Description has all 8 sections from the story template
- [ ] Every endpoint lists its allowed roles
- [ ] Acceptance criteria can each be checked by a test or a reviewer
- [ ] Scope lists what must NOT be built or modified
- [ ] Patient-facing behaviour is identified, and the story asks for a feature flag and a rollback section
- [ ] No real patient data, credentials or secrets anywhere in the ticket
- [ ] Labels and assignee are empty (the automation is the only trigger)

## 4. Environment baseline (checked before any run)

| Setting | Required value |
|---|---|
| Devin automation | `KAN - Issue created -> Smart Hospital`, Active |
| Automation trigger | Jira **Status changed** to *Ready for Devin*, project UST-HealthCare (fallback: Issue created) |
| Automation instructions | Start with `!smart_hospital__analyze_plan__build` (playbook v2) |
| Automation MCPs | Jira **All projects** + **atlassian** (Atlassian MCP) ticked |
| Network policy | `git-manager.devin.ai`, `pypi.org`, `files.pythonhosted.org` |
| GitHub connection | Automation scope **All connected repos** (the repository is public) |
| Atlassian account | Linked to the Devin user (profile -> Settings -> Connections) |
| Branch protection on `main` | PR required, 1 approval, `CI` checks required |

## 5. Rollback rules for patient-facing changes

Design rules (checked at CP1 and CP4):

1. New patient-facing behaviour sits behind a feature flag read from configuration, **default OFF**.
2. Migrations are **expand, then contract**: a release only adds (nullable columns, new tables).
   Making a column required, renaming or dropping happens in a later release.
3. Every migration has a working `downgrade`; CI runs upgrade -> downgrade -> upgrade.
4. Rollback never deletes patient data without explicit authorization.

Rollback layers, in order:

| Layer | Action | Use when | Decides |
|---|---|---|---|
| 1. Feature flag | Set the flag OFF in configuration | First response to any trigger | Release owner |
| 2. Code revert | `git revert -m 1 <merge-commit>` on a branch -> PR -> CI -> approval -> merge. Never force-push `main` | Flag cannot contain the problem | Release owner + reviewer |
| 3. Migration step-back | `alembic downgrade -1`, only if the new schema breaks the old code | Schema fails after the revert | Tech lead, after a data-loss check |
| 4. Data repair | Reviewed script in its own PR | Bad data was written while live | Tech lead + data owner |
| 5. Close-out | Jira incident ticket, affected records noted, regression test, workflow fix if a checkpoint missed it | Always | Ticket owner |

Triggers: a failed smoke test; error rate on changed endpoints above the agreed threshold; any patient data
shown to the wrong role or written to logs (roll back immediately).

## 6. Validation on a second story

The workflow counts as reusable only after it runs **unchanged** on a story it was not built from.

- Built from: KAN-6 Patient Service.
- Validation story: Appointment Service (phase 6) — patient-facing, concurrency-sensitive, different domain.
- Every human action must happen at a named checkpoint. Anything else is logged as a workflow defect and
  fixed here (version bump), not patched into the one story.
- Validated when all six checkpoints produce evidence and every unplanned intervention has a merged
  workflow fix.

## 7. Change control

This file is versioned with the code. Change it through a pull request; record the version used in each
story's PR description.

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-10-19 | Initial workflow derived from KAN-6; six checkpoints, rollback rules, second-story validation |
