# 04 — `Result<T>`, ProblemDetails, validation filter

Status: ready-for-agent
Depends on: 03

## Goal

One error contract, used by every endpoint, before any endpoint exists.

## Scope

- `Result<T>` with outcomes: `Success`, `NotFound`, `ValidationError`, `Conflict`,
  `DependencyUnavailable`, `Unexpected`. A `DependencyUnavailable` carries *which* dependency.
- `ToActionResult()` extension mapping outcomes to status codes: `200`/`201`/`202`/`204`,
  `400`, `404`, `409`, `503`, `500`. Controllers never construct a `Result<T>` and never inspect one.
- RFC 9457 `ProblemDetails` for every failure, with distinct `type` and `title` per cause. A broker
  outage and a database outage are both `503` and are **not** the same problem type.
- Global exception handler producing `ProblemDetails`; `detail` and `traceId` only outside
  Production.
- FluentValidation 12.2.0 with an MVC action filter mapping failures to `ValidationProblemDetails`.
  `FluentValidation.AspNetCore` is deprecated and is not referenced.
- Middleware order: request logging → exception handler → status code pages → routing → rate
  limiter → controllers.

## Acceptance criteria

- [ ] Every error response is `ProblemDetails`; no endpoint returns a bare string or an anonymous object
- [ ] Broker-unavailable and database-unavailable produce different `type` values
- [ ] A validator failure produces `400` with `ValidationProblemDetails` listing each field
- [ ] No controller contains `try`/`catch`
- [ ] Unit tests cover the outcome-to-status mapping

## Gaps closed

GAP-32 (no error contract), GAP-35 (broad catches merging unrelated causes).
