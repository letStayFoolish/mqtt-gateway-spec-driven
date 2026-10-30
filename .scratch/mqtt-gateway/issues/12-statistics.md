# 12 — Rolling-window statistics in Redis

Status: ready-for-agent
Depends on: 10

## Goal

Statistics that mean what they claim, and that never take the service down.

## Scope

- Per the design in [ADR-0004](../../../docs/adr/0004-redis-rolling-window.md):
  - `stats:{topic}:{yyyyMMddHHmm}`, TTL 70 minutes
  - `INCR` and `EXPIRE` in one pipelined transaction; `EXPIRE` only when `INCR` returns 1
  - active concrete topics tracked in the `stats:topics` set
  - reads are a single `MGET` over the last 60 buckets
- `GET /api/stats` — total plus per-topic counts, with `windowMinutes` stated in the response.
- `GET /api/stats/topic?filter=` — a query parameter, because topics contain `/`.
- `IConnectionMultiplexer` registered as a lazily-connected singleton. Never
  `ConnectionMultiplexer.Connect` synchronously in a constructor at startup.
- Redis unavailable → ingest and `/api/messages` unaffected, `/api/stats` returns `200` with
  `"degraded": true` and zeroed counts. Never `500`.
- Unit tests for bucket-key arithmetic across an hour boundary and across midnight.

## Acceptance criteria

- [ ] With a shortened bucket TTL, counts age out progressively rather than all at once —
      demonstrating a rolling window, not a sliding expiry
- [ ] Subscribing to `sensors/#` and publishing to `sensors/a` and `sensors/b` makes both appear in
      `GET /api/stats`
- [ ] `GET /api/stats/topic?filter=sensors/room1/temp` returns a count — a topic containing `/`
      is not a routing problem
- [ ] Stopping Redis leaves ingest and `/api/messages` working; `/api/stats` reports `degraded`
- [ ] A crash between `INCR` and `EXPIRE` cannot leave a key without a TTL

## Gaps closed

GAP-03 (route cannot bind a topic containing `/`), GAP-18 (write-only Redis keys nothing reads),
GAP-21 (sliding expiry mislabelled as a rolling window), GAP-22 (non-atomic `INCR` + `EXPIRE`),
GAP-23 (wildcard counters invisible to the stats view), GAP-09 (hardcoded, synchronously connected
Redis).
