# 08 — Subscriptions CRUD with database-enforced uniqueness

Status: ready-for-agent
Depends on: 05, 07

## Goal

Subscriptions are durable, unique, and applied at the broker.

## Scope

- `GET /api/subscriptions` — from the database, which is the source of truth.
- `POST /api/subscriptions` with `topicFilter` and `qos` → `201` with `Location`.
- `DELETE /api/subscriptions/{id:int}` → `204`, removes the row and unsubscribes at the broker.
- **Topic filter** validation — distinct from topic names: `+` and `#` permitted, `#` only as the
  final level, empty rejected, `$`-prefixed rejected, 512-byte cap.
- Ordering: persist first, then subscribe at the broker. If the broker call fails, the row is rolled
  back, so the database never claims a subscription the broker does not have. The opposite ordering
  leaves an orphaned broker subscription that no restart will reconcile.
- The unique violation surfaced by the provider maps to `409`, never `500`.
- Database unavailable → `503` with the database problem type; broker unavailable → `503` with the
  broker problem type. The two are distinguishable.

## Acceptance criteria

- [ ] Eight concurrent `POST`s of the same filter: one `201`, seven `409`, exactly one row
- [ ] `sensors/#/temp` returns `400` — `#` is not the final level
- [ ] `sensors/+/temp` is accepted
- [ ] `DELETE` stops delivery for that filter, verified by publishing afterwards
- [ ] Broker-down and database-down produce different problem types

## Gaps closed

GAP-15 (no per-subscription QoS), GAP-16 (no unsubscribe), GAP-26 (racy duplicate check).
