# MG-015 — OpenTelemetry traces, metrics, dashboard

Depends on: MG-010, MG-012

## Goal

When the service misbehaves, the cause is visible without attaching a debugger.

## Scope

- OpenTelemetry tracing and metrics exported over OTLP to the Aspire Dashboard container from
  MG-002. Instrumentation for ASP.NET Core, HttpClient, EF Core, and StackExchange.Redis.
- Custom spans so one message is traceable end to end: `POST /api/publish` → broker → ingest
  handler → channel → batch insert → Redis increment.
- Custom metrics:
  - `mqttgateway.ingest.channel_depth` (gauge) — the backpressure signal from ADR-0003
  - `mqttgateway.ingest.batch_size` (histogram)
  - `mqttgateway.ingest.latency` (histogram) — receipt to persisted
  - `mqttgateway.mqtt.reconnects` (counter)
  - `mqttgateway.ingest.rejected_payloads` (counter)
  - `mqttgateway.ratelimit.rejections` (counter)
- Serilog enriched with trace and span identifiers so a log line links to its trace.
- Payloads are never logged at `Information`.

## Acceptance criteria

- [ ] A published message produces one connected trace across API, ingest, database, and Redis
- [ ] Channel depth is visible on the dashboard and rises under a slowed database
- [ ] Log entries carry `TraceId`
- [ ] No payload content appears in logs at `Information`

## Gaps closed

GAP-20 (no monitoring), and makes the ADR-0003 backpressure trade observable rather than assumed.
