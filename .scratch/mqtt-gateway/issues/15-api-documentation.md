# 15 — API documentation

**What to build:** A developer can discover and exercise every endpoint without reading the source,
and can see each endpoint's failure responses next to its success response.

**Blocked by:** 10, 11

**Status:** ready-for-agent

- [ ] A machine-readable API document is served and is valid
- [ ] An interactive reference is available in development and absent in production
- [ ] Every endpoint documents its real status codes, including dependency-unavailable and throttled
      responses, and the problem shapes they carry
- [ ] A request collection covers every endpoint, including the failure cases — invalid topic,
      duplicate filter, oversized payload, throttled request
- [ ] Every request in that collection succeeds against a running stack
- [ ] Documentation comments are written only where the runtime surfaces them to a caller in the API
      document, or where logic is genuinely non-obvious
