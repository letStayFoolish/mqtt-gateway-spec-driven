# ADR-0004 — Rolling-window statistics in Redis

Date: 2026-10-30 · Status: Accepted

## Context

The requirement is live traffic statistics per topic over a **rolling one-hour window**.

The implementation this naturally attracts — and the one the original assignment specifies — is
`INCR stats:{topic}:count` followed by `EXPIRE stats:{topic}:count 3600` on every message. It does
not produce a rolling window. Each increment pushes the expiry forward, so the key survives as long
as traffic continues and the counter accumulates from the first message ever seen. The window it
actually implements is "everything since the beginning, discarded after an hour of silence." For an
actively publishing topic — the only interesting case — the reported figure is wrong and the label
`"timeWindow": "1 hour"` is false.

Two further defects travel with it. `INCR` and `EXPIRE` are separate commands, so a crash between
them leaves a key with no expiry at all, forever. And when the subscription is a wildcard such as
`sensors/#`, counters are created against *concrete* topics, while the stats endpoint looks up the
filter — returning zero for every topic that has traffic.

## Options

**Per-minute buckets.** `stats:{topic}:{yyyyMMddHHmm}`, TTL 70 minutes. A read is one `MGET` over
the last 60 keys, summed. Resolution of one minute; cost per read is constant and independent of
traffic volume.

**Sorted set per topic**, timestamps as scores, trimmed with `ZREMRANGEBYSCORE`. Resolution of one
second, at the cost of one set member per message and a trim on every read or write. Memory grows
with traffic rather than with time.

## Decision

Per-minute buckets.

- Counter key: `stats:{topic}:{yyyyMMddHHmm}`, TTL 70 minutes.
- Increment and expiry issued as one pipelined transaction; `EXPIRE` only when `INCR` returns 1.
- Active concrete topics tracked in a Redis set `stats:topics` so the stats endpoint enumerates
  what actually has traffic rather than what someone subscribed to. `SCAN` in a loop is not used.
- A read is a single `MGET` across the 60 bucket keys for the window.

## Consequences

- The window is genuinely rolling and the reported `windowMinutes` is true.
- Resolution is one minute. For "traffic per topic" that is ample; nothing in the requirements asks
  for finer, and a sorted set's per-message memory cost is not worth buying resolution nobody reads.
- Bucket boundaries mean the window edge moves in one-minute steps rather than continuously. The
  response states `windowMinutes` so a caller is never guessing.
- Memory is bounded by (topics × 70) keys regardless of message volume.
- **Redis is derived state.** When it is unavailable, ingest continues, `/api/messages` is
  unaffected, `/api/stats` returns `200` with `"degraded": true`, and `/health/ready` reports
  `Degraded`. Statistics are never a reason to return `500`.
- Database and Redis can disagree after a partial failure. There is no distributed transaction and
  there will not be one; the inconsistency is named in the README rather than hidden.
