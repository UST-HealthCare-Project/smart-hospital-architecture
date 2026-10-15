# Smart Hospital Repository Rules

These detailed rules complement `AGENTS.md`. Keep `AGENTS.md` as the concise entry point.

## Python and API style
- Use type hints on public functions and important interfaces.
- Use Pydantic request/response schemas; do not expose ORM objects directly as API contracts.
- Keep route handlers thin. Put business rules in service/use-case functions.
- Use dependency injection for database sessions, authenticated users and service clients.
- Use consistent HTTP semantics: 401 unauthenticated, 403 forbidden, 404 missing resource, 409 domain conflict, 422 invalid input.
- Avoid broad exception handlers that hide failures, unnecessary abstractions and global mutable state.

## Data and database
- Use SQLAlchemy 2.x patterns consistent with the repository.
- Use Alembic migrations for schema changes and inspect generated migrations.
- Add database constraints for uniqueness and integrity invariants.
- Use transactions for multi-step writes. Booking conflict protection must remain correct under concurrent requests.
- Handle timestamps as timezone-aware UTC. Interpret facility schedules using an explicit IANA timezone.
- Never share tables across microservices. Cross-service references are IDs validated through service contracts.
- Do not hard-delete records casually; follow the service's documented lifecycle and retention assumptions.

## Authentication and authorization
- Hash passwords with a maintained password-hashing library.
- Validate JWT signature, expiry, issuer and audience according to configuration.
- Enforce permissions server-side for every protected operation; default to deny when unclear.
- Do not trust role, user ID or facility access claims supplied in request bodies.
- Keep secrets in environment variables or an approved secret manager. `.env.example` contains placeholders only.
- Never commit tokens, private keys, real passwords or credentials in logs, fixtures or docs.

## Privacy and healthcare scope
- Use synthetic data only. Avoid real patient, contact, date-of-birth, medical-record or client data in the repository.
- Do not log patient-identifying fields, full payloads, encounter notes or authorization headers.
- Use least-privilege response schemas and role-based access for visit information.
- Default notifications to generic content without clinical details.
- This is a demo administrative system: no diagnosis, treatment recommendation, clinical decision automation or emergency triage.
- Do not assert regulatory compliance without formal review.

## Tests and verification
- Add unit tests for domain logic and API tests for endpoint behavior.
- Cover success, invalid input, missing resource, authorization, conflicts and state transitions.
- Use test-only dependency overrides/mocks for unavailable services. Never weaken production auth to simplify tests.
- Run tests and report exact commands, exit status and known gaps. Do not claim tests passed if not run.
- Add regression tests for defects and concurrency tests for booking invariants where practical.

## Cross-service integration
- Use versioned HTTP contracts or explicit domain events.
- Define request/response schemas and timeouts; make clients mockable.
- No cross-service database joins or direct writes.
- Document failure and retry behavior. Use idempotency for repeatable booking/notification processing where appropriate.
- Do not introduce a message broker unless requested and justified by the current phase.

## Dependencies, docs and changes
- Follow existing dependency pinning. Explain new runtime dependencies.
- Avoid broad upgrades and unrelated refactors.
- Update service README, `.env.example`, OpenAPI description and migrations as appropriate.
- Never overwrite user changes or run destructive commands without explicit permission.
- Keep diffs small and reviewable.

## Done criteria
A change is done only when requirements/assumptions are stated, service boundaries and authorization are respected, relevant tests are run or limitations documented, migrations/config/docs are updated where applicable, the diff is reviewed, and actual validation results are reported.
