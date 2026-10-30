# Tickets

Work is delivered one ticket at a time. Each ticket is a branch (`feat/MG-00X-short-name`), merged
to `main` when its acceptance criteria hold.

Every ticket states what it closes from [`../spec-gaps.md`](../spec-gaps.md). A gap is only closed
when the behaviour is asserted by a test naming its ID.

| ID | Title | Depends on |
|---|---|---|
| [MG-001](MG-001-repository-skeleton.md) | Repository skeleton, solution, central packages, CI | — |
| [MG-002](MG-002-infrastructure-compose.md) | Infrastructure compose: EMQX, MariaDB, Redis, telemetry | MG-001 |
| [MG-003](MG-003-configuration-and-logging.md) | Typed configuration, Serilog, startup validation | MG-001 |
| [MG-004](MG-004-result-and-problem-details.md) | `Result<T>`, ProblemDetails, validation filter | MG-003 |
| [MG-005](MG-005-mqtt-connection.md) | MQTT connection lifecycle, reconnect, LWT, `/api/connection` | MG-003, MG-004 |
| [MG-006](MG-006-publish-endpoint.md) | `POST /api/publish` and topic-name validation | MG-004, MG-005 |
| [MG-007](MG-007-persistence-foundation.md) | `GatewayDbContext`, entities, migrations, migrator | MG-003 |
| [MG-008](MG-008-subscriptions-crud.md) | Subscriptions CRUD with database-enforced uniqueness | MG-005, MG-007 |
| [MG-009](MG-009-subscription-restoration.md) | Restoration barrier gating readiness | MG-008 |
| [MG-010](MG-010-ingest-pipeline.md) | Channel-based ingest with backpressure | MG-007, MG-009 |
| [MG-011](MG-011-messages-query.md) | `GET /api/messages` with keyset paging | MG-010 |
| [MG-012](MG-012-statistics.md) | Rolling-window statistics in Redis | MG-010 |
| [MG-013](MG-013-health-checks.md) | `/health/live` and `/health/ready` | MG-009, MG-012 |
| [MG-014](MG-014-rate-limiting.md) | Token-bucket limiter on publish | MG-006 |
| [MG-015](MG-015-observability.md) | OpenTelemetry traces, metrics, dashboard | MG-010, MG-012 |
| [MG-016](MG-016-retention-purge.md) | Configurable message retention | MG-011 |
| [MG-017](MG-017-integration-tests.md) | Integration suite: all nine scenarios | MG-013 |
| [MG-018](MG-018-containerise-app.md) | Dockerfile and application in compose | MG-013 |
| [MG-019](MG-019-api-documentation.md) | OpenAPI document, Scalar UI, `.http` file | MG-013 |
| [MG-020](MG-020-readme.md) | README and known limitations | MG-018 |
