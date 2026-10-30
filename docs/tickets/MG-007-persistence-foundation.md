# MG-007 — `GatewayDbContext`, entities, migrations, migrator

Depends on: MG-003

## Goal

A schema that matches spec §5, applied reproducibly, including inside the container stack.

## Scope

- `GatewayDbContext` with `Message` and `Subscription`, configured through
  `IEntityTypeConfiguration<T>` living beside each entity in its feature folder. No data
  annotations on entities.
- Columns, types, and indexes exactly as spec §5, including `varchar(512)` on `topic` and
  `topic_filter`, `varbinary` payload, `datetime(6)` UTC timestamps, the unique index on
  `topic_filter`, and `ix_messages_topic_received_at`.
- Provider registration with an explicit `MariaDbServerVersion` — **not** `ServerVersion.AutoDetect`,
  which opens a connection during service registration and makes startup depend on the database.
- Initial migration. The generated `Up()` is read before it is applied.
- A one-shot `migrator` compose service applying migrations. Not `Database.Migrate()` at API
  startup, which races when more than one instance starts.

## Acceptance criteria

- [ ] `dotnet ef migrations add` and `database update` produce the schema in spec §5
- [ ] `topic` is `varchar(512)`, not `longtext` — verified against the live schema
- [ ] Two concurrent inserts of the same `topic_filter`: one succeeds, one raises a unique violation
- [ ] The API starts with the database **down** and reports not ready, rather than crashing
- [ ] Applying migrations twice is a no-op

## Gaps closed

GAP-07 (`AutoDetect` plus `depends_on`), GAP-08 (no migration path in Docker), GAP-26 (racy
duplicate check), GAP-27 (`longtext` cannot be indexed), GAP-28 (`DateTime.Now`), GAP-30 (payload
type), GAP-38 (unmaintained provider).
