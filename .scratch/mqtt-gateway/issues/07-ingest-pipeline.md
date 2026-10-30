# 07 — Ingest pipeline

**What to build:** Every message the gateway receives is written down and can be read back over
HTTP. A message is acknowledged to the broker only after it has been stored, so the broker is never
told a message was handled when it was lost. When storage is slow, collection slows down rather than
losing data.

**Blocked by:** 05

**Status:** ready-for-agent

- [ ] A message published to a subscribed topic is persisted and readable through a message query
- [ ] Integration test: a thousand messages at least-once QoS produce at least a thousand stored
      rows, with no error in the logs
- [ ] The receive handler does no storage work; it stamps the receipt time **once** and hands the
      record off
- [ ] Hand-off is through a bounded buffer that blocks when full, so pressure reaches the broker as
      backpressure rather than becoming a memory leak or silent loss
- [ ] Acknowledgement is deferred until after the batch is stored
- [ ] Records are written in batches bounded by size and by a time window, with one database scope
      per batch
- [ ] No long-lived component holds a request-scoped database context; scope validation is enabled
      in development and passes
- [ ] Integration test: a deliberately slowed database slows ingestion without losing messages and
      without a concurrency error
- [ ] Integration test: shutting down mid-flight stores what was queued rather than dropping it
- [ ] Integration test: an oversized payload is rejected and counted without stalling the pipeline
- [ ] Payloads are stored as bytes; nothing assumes they are text
- [ ] No thread-pool offloading of work that is already asynchronous, and no parallel composition of
      a storage write with a cache write
