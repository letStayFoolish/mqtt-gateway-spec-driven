# 17 — Integration suite: all nine scenarios

Status: ready-for-agent
Depends on: 13

## Goal

The claims in this specification are asserted against real dependencies, not assumed.

## Scope

- `WebApplicationFactory` plus Testcontainers: MariaDB 12.3, Redis 8.10.3, and EMQX 6.3.2 built
  with `ContainerBuilder` and an `emqx ctl status` wait strategy.
- Containers shared per collection fixture, not per test class, so startup is paid once.
- Test configuration overrides connection strings **before** the DI container is built.
- Asynchronous outcomes awaited by polling with a timeout. `Thread.Sleep` is banned.
- State cleaned between tests; no execution-order dependence.
- The nine scenarios from spec §10, each naming the gap it closes:

| # | Scenario | Gap |
|---|---|---|
| 1 | Publish → broker → ingest → visible via `GET /api/messages` | GAP-17 |
| 2 | Duplicate subscription filter → `409`, exactly one row | GAP-26 |
| 3 | Broker killed → not ready → restarted → reconnect, restore, resume, no process restart | GAP-19, GAP-29 |
| 4 | Redis stopped → stats degraded, ingest and messages unaffected | GAP-18, GAP-25 |
| 5 | Connection severed without DISCONNECT → LWT delivered | GAP-12 |
| 6 | 1000 messages at QoS 1 → 1000 rows and statistics agree | GAP-02, GAP-21 |
| 7 | Two instances, same `ClientId` → session takeover observed | GAP-33 |
| 8 | Non-UTF-8 payload round-trips as base64 | GAP-30 |
| 9 | Oversized payload rejected and counted, pipeline unaffected | GAP-30 |

## Acceptance criteria

- [ ] All nine pass locally and in CI
- [ ] No `Thread.Sleep` anywhere in the test projects
- [ ] The suite passes when run twice in a row without manual cleanup
- [ ] Deleting the body of a method under test makes at least one test fail — spot-checked on
      publish, ingest, and restoration

## Gaps closed

GAP-13 (integration tests named but never specified), GAP-06 (tests that could not be written as
prescribed), GAP-39 (mocking library).
