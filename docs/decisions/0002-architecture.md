# ADR-0002 — Vertical slices in a single project

Date: 2026-10-30 · Status: Accepted

## Context

The house standard for ASP.NET Core services is Clean Architecture: `Api`, `Application`,
`Domain`, `Contracts`, and one `Infrastructure.*` project per external I/O concern, with a
mandatory repository and unit-of-work layer over EF Core.

That standard exists for services with domain logic worth protecting from infrastructure churn —
where an entity has invariants, a workflow spans aggregates, and the cost of indirection buys
genuine isolation.

This service has none of that. Its entities are records of things that happened. Its "business
logic" is: validate a topic string, write a row, increment a counter. What is actually hard here is
lifetimes, concurrency, failure isolation, and operability — and no arrangement of projects helps
with any of those.

## Decision

One project, `MqttGateway.Api`, organised by feature:

```
Features/{Publishing,Subscriptions,Messages,Statistics,Diagnostics}/
Infrastructure/{Mqtt,Persistence,Redis,Http,Options}/
```

A feature owns its controller, DTOs, validator, entity, and EF configuration. `Infrastructure/`
holds only what is genuinely shared.

No repository or unit-of-work abstraction. `DbContext` is already both.

## Consequences

- Duplication between features is accepted until it hurts. That is the trade vertical slices make:
  slice independence, paid for in DRY.
- A project boundary only earns its place when it prevents a reference we do not want. Here it
  would prevent nothing, since every feature depends on `Infrastructure` anyway.
- `IMessageRepository` with exactly one implementation would be an abstraction introduced to
  satisfy a single caller — a finding under our own pattern guidance, not a benefit.
- Tests reach `internal` types via `InternalsVisibleTo`, so a public surface does not have to be
  widened for testability.
- One open question is answered deliberately: the `Message` entity is written by MQTT ingest and
  read by the `Messages` feature. It belongs to `Messages`, which owns its schema and its queries;
  ingest depends on that feature rather than the reverse. The alternative — a shared `Domain`
  project holding entities for everyone — is the hidden layer this decision exists to avoid.
- If this service later grows real domain logic, the migration path is to extract that logic into a
  project of its own. Slices make that extraction local.
