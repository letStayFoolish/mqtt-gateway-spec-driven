# 03 — Typed configuration, Serilog, startup validation

Status: ready-for-agent
Depends on: 01

## Goal

The application starts, logs structurally, exposes trivial health endpoints, and refuses to start
on invalid configuration.

## Scope

- Options classes with `required ... { get; init; }`, bound and validated with
  `.ValidateDataAnnotations().ValidateOnStart()`: `MqttOptions`, `IngestOptions`,
  `RetentionOptions`, `RedisOptions`, `RateLimitingOptions`.
- Consumed as `IOptions<T>` through primary constructors. `IConfiguration` is never injected into
  a service.
- Serilog from configuration: console in Development, structured JSON elsewhere, plus a rolling
  file sink. Request logging registered first in the pipeline.
- Trivial `/health/live` and `/health/ready` so the shape exists from the start (13 gives them
  meaning).
- Local secrets via `dotnet user-secrets`; `appsettings.json` contains no credentials.

## Acceptance criteria

- [ ] A missing or malformed required option fails the process at startup with a message naming the
      offending key — not on the first request
- [ ] Logs carry named properties, never interpolated strings
- [ ] `GET /health/live` returns `200`
- [ ] `appsettings.json` contains no secret

## Gaps closed

GAP-11 (undefined `MqttBroker` format), GAP-44 (secrets), and the absence of any logging
specification for the API in the original assignment.

## Traps

Serilog templates use named placeholders — `_logger.LogInformation("Subscribed to {TopicFilter}", filter)`.
Interpolation defeats structured logging and is treated as a defect.
