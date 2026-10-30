# MG-009 — Restoration barrier gating readiness

Depends on: MG-008

## Goal

A connected gateway with no subscriptions is never reported as working.

## Scope

- On every successful connect, re-establish every subscription in the database at its stored QoS,
  **before** reporting readiness.
- `SubscriptionsRestored` is a distinct piece of state, exposed by `GET /api/connection` and
  consumed by `/health/ready`.
- While restoration is incomplete: `/health/ready` is `Unhealthy`.
- If restoration fails: disconnect and retry the whole set with backoff. Partial restoration is not
  a state the system stays in — an operator cannot tell which filters are missing.
- Restoration runs on *every* connect, not once at startup.

## Acceptance criteria

- [ ] Stop the broker, start it again: subscriptions are re-established without a process restart,
      and messages flow again
- [ ] During restoration `/health/ready` is `Unhealthy` and `subscriptionsRestored` is `false`
- [ ] Forcing a restoration failure disconnects and retries rather than serving traffic
- [ ] Adding a subscription, restarting the process, and publishing delivers the message —
      without re-subscribing through the API

## Gaps closed

GAP-29 (auto-subscribe ran only once at startup), GAP-19 (no reconnect), and the silent-failure
mode where `IsConnected` is true with zero subscriptions.
