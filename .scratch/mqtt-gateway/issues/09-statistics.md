# 09 — Statistics

**What to build:** An operator can see live traffic counts per topic and overall, over a window that
means what it says. When the cache is down, statistics degrade visibly instead of failing — and
nothing else is affected.

**Blocked by:** 07

**Status:** ready-for-agent

- [ ] Per-topic and total counts are available over a rolling one-hour window, and the response
      states its own window length
- [ ] Integration test: counts age out progressively rather than all at once, demonstrating a
      rolling window rather than a sliding expiry
- [ ] The topic filter is a query parameter; a topic containing slashes is not a routing problem
- [ ] Integration test: subscribing to a wildcard and publishing to several concrete topics makes
      each of them visible in the statistics
- [ ] The counter increment and its expiry are applied as one atomic unit, so a crash between them
      cannot leave a key without an expiry
- [ ] Reading a topic's window is a single round trip, not a loop over keys
- [ ] The keyspace is never scanned to discover active topics
- [ ] Integration test: stopping the cache leaves ingestion and message queries unaffected, and the
      statistics endpoint succeeds with a degraded marker rather than failing
- [ ] The cache client is connected lazily and never synchronously during startup
- [ ] Unit tests cover bucket arithmetic across an hour boundary and across midnight
