# 08 — Message query

**What to build:** A caller can read a device's history — filtered by topic and by time window,
paginated so that a large result set can be walked without seeing a row twice or missing one, and
bounded so that no single request can take the service down.

**Blocked by:** 07

**Status:** ready-for-agent

- [ ] Messages can be filtered by topic and by time range
- [ ] A single message can be retrieved by its identifier, with a not-found response when absent
- [ ] Pagination is deterministic across rows sharing a timestamp — integration test walks such a
      set and sees every row exactly once
- [ ] The page size has a ceiling; an absurd request is rejected rather than served
- [ ] Each payload is returned with an explicit statement of whether it is text or binary
- [ ] Integration test: a non-UTF-8 payload round-trips without corruption and is reported as binary
- [ ] Reads are untracked and project only the columns they need; a listing does not load payload
      blobs it will not return
- [ ] Filtering and ordering happen in the database, never in memory
- [ ] The query uses the covering index, verified against the live query plan
- [ ] History survives a restart and remains queryable
- [ ] Unit tests cover pagination cursor encoding and decoding
