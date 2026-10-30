# MG-019 — OpenAPI document, Scalar UI, `.http` file

Depends on: MG-013

## Goal

A caller can discover and exercise the API without reading the source.

## Scope

- `Microsoft.AspNetCore.OpenApi` producing an OpenAPI 3.1 document at `/openapi/v1.json`. It is a
  package reference, not part of the shared framework.
- `Scalar.AspNetCore` UI at `/scalar`, enabled in Development only. ASP.NET Core 10 ships no
  built-in UI, and Swashbuckle is no longer in the template.
- Every endpoint documents its success and failure status codes and its `ProblemDetails` shapes.
- XML documentation comments only where they add information a caller needs — .NET 10 ingests them
  into the OpenAPI document, which is the one place a comment here has a real reader.
- `MqttGateway.http` covering every endpoint, including the failure cases: invalid topic,
  duplicate subscription, oversized payload, rate-limit rejection.

## Acceptance criteria

- [ ] `/openapi/v1.json` is valid OpenAPI 3.1
- [ ] Every endpoint lists its real status codes, including `503` and `429`
- [ ] `/scalar` is reachable in Development and absent in Production
- [ ] Every request in the `.http` file succeeds against a running stack

## Gaps closed

GAP-42 (Swagger URLs and ports that cannot work on a current SDK, and a UI disabled by environment
in the documented Docker configuration).
