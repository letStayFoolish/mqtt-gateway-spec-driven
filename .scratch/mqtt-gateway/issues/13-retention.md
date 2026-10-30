# 13 — Retention

**What to build:** An operator decides how long message history is kept, so that storage growth is a
decision rather than an accident. The default leaves current behaviour unchanged.

**Blocked by:** 08

**Status:** ready-for-agent

- [ ] With retention configured, rows older than the threshold are removed and newer rows are not
- [ ] With the default configuration, nothing is ever removed
- [ ] Removal happens in bounded batches on an interval, so clearing a large backlog does not block
      ingestion beyond a single batch
- [ ] The worker survives a database outage and resumes afterwards
- [ ] Rows removed are exposed as a metric
