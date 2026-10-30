# 14 — Application in the container stack

**What to build:** A clone and one command produce a working system on a machine that has nothing
but a container runtime — application included, schema applied, dependencies waited for.

**Blocked by:** 10

**Status:** ready-for-agent

- [ ] Bringing the stack up from a clean checkout yields a working system with no manual step
- [ ] A readiness request against the published port succeeds once dependencies are healthy
- [ ] A full cycle — publish, ingest, query, statistics — works entirely inside the stack
- [ ] The application waits for dependencies to be genuinely ready, not merely started
- [ ] Migrations are applied by a dedicated one-shot step before the application accepts traffic
- [ ] The published port matches the port the runtime actually listens on
- [ ] Service names are used as hostnames inside the network; no loopback address appears in any
      inter-service setting
- [ ] The image is built in stages, ships without build tooling or source, and runs as a non-root
      user
- [ ] Editing a source file and rebuilding does not re-run dependency restore
- [ ] Build context excludes build output, version-control metadata, and test artefacts
- [ ] All configuration arrives through environment variables; no secret is in the compose file or
      baked into the image
- [ ] Taking the stack down and up preserves messages and subscriptions
