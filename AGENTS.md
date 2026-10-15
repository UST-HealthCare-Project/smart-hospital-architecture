# AGENTS.md — Smart Hospital & Patient Management System

## Project purpose
Build a development/demo Smart Hospital & Patient Management System as eight independently testable backend microservices. Use Python FastAPI, Pydantic, SQLAlchemy, Alembic and PostgreSQL. Work incrementally, one explicitly requested microservice or feature at a time.

This is an administrative demo, not a certified clinical system. Use synthetic data only. Do not implement diagnosis, treatment recommendations, emergency triage or autonomous clinical decisions.

## Repository workflow
- Read this file and `Smart_Hospital_Repository_Rules.md`, the root README, target service README, API contracts, dependency manifests and tests before editing.
- Before editing, present a concise plan: assumptions, affected files, API/data changes, dependencies, security implications and tests.
- Implement only the requested phase. Do not start the next microservice automatically.
- Preserve existing conventions and unrelated work. Keep changes small and reviewable.
- Prefer Python 3.12+, FastAPI, Pydantic v2, SQLAlchemy 2.x, Alembic, PostgreSQL, pytest and HTTPX TestClient, unless compatible versions are already pinned.
- Use layered structure: routers, schemas, models, services/use-cases, repositories, configuration and tests. Keep business logic out of route handlers.
- Use `/api/v1`, explicit request/response schemas, UUID IDs where appropriate, timezone-aware UTC timestamps, pagination, consistent HTTP errors, OpenAPI docs and a health endpoint.
- Add/update tests for each behavior change. Run targeted tests and then relevant broader tests. Report actual commands/results; never claim tests passed if not run.
- Create Alembic migrations for schema changes. Never reset/drop/delete persistent data without explicit authorization.
- Keep each service independently runnable/testable, with its own dependencies, config, migrations, tests and README where practical.

## Eight microservices and boundaries
1. Identity & Access — users, password hashes, JWTs, roles and authorization.
2. Patient — registration, demographics and patient lookup.
3. Provider — doctor/provider profiles, specialties and active status.
4. Facility & Department — facilities, departments, timezone metadata and optional rooms.
5. Scheduling & Availability — working hours, exceptions and advisory slot queries.
6. Appointment — booking lifecycle, rescheduling, cancellation and concurrency-safe booking integrity.
7. Encounter / Visit — visit records/history and restricted non-diagnostic administrative notes.
8. Notification — notification requests, templates, mock delivery adapters and delivery status.

- Each service owns its data, tables and migration history. Never directly query/write another service's database tables.
- Use explicit API contracts or domain events for cross-service communication. Keep clients mockable in tests.
- If a dependency service is unfinished, define a narrow contract and mock it in tests. Do not invent undocumented endpoints or add production authorization bypasses.
- Shared contracts may contain API/event schemas only, not shared business/database models that tightly couple services.
- Scheduling availability is advisory. Appointment owns final booking correctness and must use transaction/constraint/locking logic that prevents concurrent double-booking; a pre-check alone is insufficient.
- Notification should be transport-independent. Start with a mock adapter and durable request/status model; add a broker only when explicitly requested and justified.
- API Gateway, React frontend, Docker orchestration, Kubernetes, cloud deployment, CI/CD and observability are future scope unless explicitly requested.

## Security and privacy
- Use fictional/synthetic patient, provider and contact data in development, tests, demos and screenshots.
- Never commit `.env`, passwords, API keys, JWT signing keys, private keys or real credentials. Commit `.env.example` with placeholders only.
- Never log passwords, authorization headers, tokens, full request bodies, patient contact details, dates of birth or visit-note contents.
- Hash passwords with a reputable library; never store plaintext passwords.
- Authentication is not authorization. Enforce permission checks server-side on every protected operation. Never trust roles supplied by the caller.
- Verify JWT signature, expiry, issuer and audience as configured.
- Return only the minimum fields needed by each API response. Avoid clinical details in notifications.
- Do not claim HIPAA/GDPR or other regulatory compliance from code alone; production use requires qualified review.

## Testing and quality
- Cover success paths, invalid payloads, missing resources, unauthorized/forbidden requests and relevant edge cases.
- Use isolated test configuration, synthetic fixtures and mocks for external services.
- Add regression tests for fixes. For booking, test conflict/concurrency behavior where practical.
- Do not claim coverage or passing tests without evidence. Report skipped tests and known gaps.
- Inspect diffs for secrets, debug code, unsafe logs, accidental generated files, unrelated edits and missing migrations.

## Required completion report
Report: implementation summary; files changed; API/schema/migration changes; commands run and actual test results (`PASS`, `FAIL`, `NOT RUN`); security/boundary review; assumptions/limitations; and confirmation that you stopped at the requested phase.

## Stop conditions
- Do not start another microservice or future phase unless explicitly asked.
- Ask before adding brokers, external providers, real email/SMS, production infrastructure, broad dependency upgrades or destructive operations.
- If failures appear pre-existing, capture evidence and report them rather than making unrelated edits.
