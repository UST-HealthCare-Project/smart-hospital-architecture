# Smart Hospital & Patient Management System

Development-demo backend for a Smart Hospital & Patient Management System, built phase by phase as
independently testable Python FastAPI microservices backed by PostgreSQL.

> This is an administrative learning/demo application, **not** a certified clinical system. Use synthetic,
> fictional data only. It does not provide diagnosis, treatment recommendations or clinical decision support.

## Services

| Phase | Service | Path | Status |
|-------|---------|------|--------|
| 1 | Identity & Access | [`services/identity-service`](services/identity-service/README.md) | Implemented |
| 2 | Patient | [`services/patient-service`](services/patient-service/README.md) | Implemented |
| 3 | Provider | [`services/provider-service`](services/provider-service/README.md) | Implemented |
| 4 | Facility & Department | [`services/facility-service`](services/facility-service/README.md) | Implemented |
| 5 | Scheduling & Availability | — | Planned |
| 6 | Appointment | — | Planned |
| 7 | Encounter / Visit | — | Planned |
| 8 | Notification | — | Planned |

Each service owns its own database, tables and Alembic migration history. Services never query or write
another service's tables directly; they integrate through explicit HTTP API contracts or events.

See each service's README for setup, configuration, migration, run and test instructions.
