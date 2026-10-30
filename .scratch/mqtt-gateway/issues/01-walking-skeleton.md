# 01 — Walking skeleton

**What to build:** The service starts, answers a liveness request, and refuses to start when its
configuration is wrong. Nothing talks to a broker, a database, or a cache yet — this is the thinnest
complete path from an HTTP request to a response, plus the test host that will carry every later
slice.

**Blocked by:** None — can start immediately.

**Status:** ready-for-agent

- [ ] The application builds with no warnings and starts locally
- [ ] A liveness request succeeds
- [ ] Configuration is bound to typed options and validated at startup; a missing or malformed
      required value stops the process with a message naming the offending key, rather than
      surfacing on the first request
- [ ] Logging is structured, with named fields rather than interpolated message templates
- [ ] An integration test boots the real application through a test host and asserts the liveness
      response
- [ ] The test runner is configured so that running the suite actually works on the current SDK —
      the in-box template pairing does not
- [ ] Package versions are declared centrally, in one place
- [ ] Continuous integration restores, builds, and runs the suite on every push and pull request
- [ ] Build output never enters version control
