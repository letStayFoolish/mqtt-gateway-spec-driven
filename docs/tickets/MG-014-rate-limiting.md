# MG-014 — Token-bucket limiter on publish

Depends on: MG-006

## Goal

One caller cannot exhaust the gateway through the publish endpoint.

## Scope

- Built-in `AddRateLimiter` with a token-bucket policy applied to `POST /api/publish` only.
  Token bucket rather than fixed window because bursty publishing is the normal shape of gateway
  traffic, and a fixed window either rejects legitimate bursts or permits twice the intended rate
  across a boundary.
- Limits from `RateLimitingOptions`: token limit, replenishment period, tokens per period,
  queue limit.
- Rejections return `429` with `ProblemDetails` and a `Retry-After` header.
- Rejection count exported as a metric.
- The limiter sits after routing and before the controller, so rejected requests do no work.

## Acceptance criteria

- [ ] Exceeding the configured rate returns `429` with `ProblemDetails` and `Retry-After`
- [ ] A burst within the bucket size is accepted
- [ ] Other endpoints are unaffected
- [ ] Limits are configurable without a rebuild

## Gaps closed

Unbounded publish input — one of the three unbounded inputs named in spec §9.5.
