# 12 — Observability

**What to build:** When the service misbehaves, the cause is visible without attaching a debugger. A
single message can be followed from the HTTP request through the broker, the ingest buffer, the
batch write, and the cache, in one trace.

**Blocked by:** 07, 09

**Status:** ready-for-agent

- [ ] A published message produces one connected trace spanning the API, the broker, ingestion, the
      database, and the cache
- [ ] Traces and metrics are exported to the dashboard in the development stack with no extra setup
- [ ] The ingest buffer depth is exported as a metric and visibly rises when the database is slowed
- [ ] Batch size, ingest latency, reconnect count, rejected payloads, and throttled requests are
      exported as metrics
- [ ] Log entries carry their trace identifier
- [ ] Payload contents never appear in logs at informational level
- [ ] Instrumentation covers the HTTP surface, the database, and the cache client
