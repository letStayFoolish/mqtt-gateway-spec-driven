# MG-013 — `/health/live` and `/health/ready`

Depends on: MG-009, MG-012

## Goal

Two endpoints with genuinely different meanings, each of which an orchestrator can act on.

## Scope

- `/health/live` — the process is alive. It does **not** check dependencies. A liveness probe that
  fails during a dependency outage causes restart loops that make the outage worse.
- `/health/ready` — the service can do its job:
  - `Unhealthy`: broker disconnected, subscriptions not restored, or database unreachable
  - `Degraded`: Redis unreachable
  - `Healthy`: otherwise
- Every check is cheap and has a timeout. A health check that runs an expensive query is itself an
  availability risk.
- Response includes per-check status and duration.

## Acceptance criteria

- [ ] Stopping Redis: ready is `Degraded`, live stays `200`
- [ ] Stopping the broker: ready is `Unhealthy`, live stays `200`
- [ ] Stopping the database: ready is `Unhealthy`, live stays `200`
- [ ] Connected but subscriptions not yet restored: ready is `Unhealthy`
- [ ] No check exceeds its timeout under a stopped dependency

## Gaps closed

GAP-20 (no health, readiness, or metrics), and the silent-failure mode from MG-009.
