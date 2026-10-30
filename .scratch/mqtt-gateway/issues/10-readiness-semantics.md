# 10 — Readiness semantics

**What to build:** Liveness and readiness mean different things, so that a dependency outage takes
the service out of rotation without triggering a restart loop. One request tells an on-call engineer
which dependency is at fault.

**Blocked by:** 06, 09

**Status:** ready-for-agent

- [ ] Liveness checks the process only, and never a dependency
- [ ] Readiness is unhealthy when the broker is disconnected, when subscriptions are not restored,
      or when the database is unreachable
- [ ] Readiness is degraded — not unhealthy — when only the cache is unreachable
- [ ] Integration tests: stopping the cache, the broker, and the database each produce a different,
      distinguishable readiness result, while liveness stays healthy in all three
- [ ] Each check is cheap and time-limited; no check exceeds its timeout even when its dependency is
      stopped
- [ ] The response reports per-check status and duration
