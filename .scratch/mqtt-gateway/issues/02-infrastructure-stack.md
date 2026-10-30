# 02 — Infrastructure stack

**What to build:** One command brings up the broker, the database, the cache, and the telemetry
dashboard, all genuinely ready rather than merely started. The broker refuses anonymous clients. A
reusable container fixture lets every later slice test against real dependencies instead of
substitutes.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] Bringing the stack up from a clean state reaches a healthy state for every service, with no
      manual step
- [ ] Every image tag is pinned; no floating tag anywhere
- [ ] Each service has a health check that reflects genuine readiness, not process start
- [ ] A desktop MQTT client can connect only with credentials; an anonymous connection is refused
- [ ] Credentials come from an ignored environment file, shipped as a placeholder template; a
      missing value fails loudly rather than defaulting to empty
- [ ] No credential appears in any committed file
- [ ] Taking the stack down and bringing it back preserves database contents; cache contents are not
      preserved, which is correct for derived state
- [ ] The compose project is named distinctly, so it can never adopt another project's containers or
      share its volumes
- [ ] An integration fixture starts the real dependencies for tests and is shared across a test
      collection rather than recreated per class
- [ ] A smoke test proves all three dependencies are reachable from the test host
