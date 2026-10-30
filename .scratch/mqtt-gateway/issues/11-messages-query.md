# 11 — `GET /api/messages` with keyset paging

Status: ready-for-agent
Depends on: 10

## Goal

Messages are queryable from the database — the only source of truth — with bounded, deterministic
paging.

## Scope

- `GET /api/messages?topic=&from=&to=&cursor=&pageSize=`.
- `GET /api/messages/{id:long}` → `404` when absent.
- Keyset paging over `(received_at desc, id desc)`. The cursor is opaque and base64-encoded;
  `received_at` alone is not unique, so the composite is required for determinism.
- `pageSize` defaults to 50, capped at 200. A caller asking for 1 000 000 gets `400`.
- Reads are `AsNoTracking` and project with `.Select(...)` before materialising — the blob column is
  never loaded for a listing that does not need it.
- Filtering and ordering happen in the database, never in memory.
- Payload rendered as `utf8` or `base64` with `payloadEncoding` stating which.

## Acceptance criteria

- [ ] Messages persist across a process restart and remain queryable
- [ ] Paging through a result set with identical `received_at` values returns every row exactly once
- [ ] `pageSize=1000000` returns `400`
- [ ] A non-UTF-8 payload round-trips as `base64` without corruption
- [ ] The query uses `ix_messages_topic_received_at` — verified with `EXPLAIN`

## Gaps closed

GAP-17 (endpoint returned an in-memory list, empty after restart, unfiltered and unpaged),
GAP-30 (payload encoding).
