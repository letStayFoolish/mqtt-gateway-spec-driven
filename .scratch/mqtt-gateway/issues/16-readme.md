# 16 — README

**What to build:** Enough for someone who has never seen the repository to run it, understand it,
and know what it does not do.

**Blocked by:** 14

**Status:** ready-for-agent

- [ ] A reader who has never seen the repository can bring the stack up by following it exactly
- [ ] Both ways of running it are covered — everything in containers, and the application on the
      host against containerised dependencies
- [ ] Every configuration key is documented with its effect and its default
- [ ] How migrations are applied, and what to do when they fail
- [ ] How to run the tests, including the container runtime they require
- [ ] Links to the specification and the decision records
- [ ] Known limitations are disclosed rather than discovered: one instance per client identity;
      store and cache can diverge after a partial failure; deferred acknowledgement allows duplicate
      rows after a crash; statistics resolution and window granularity; no API authentication; the
      broker's licence terms; the ORM provider is a community fork
