# ADR-0006 — Test framework and broker under test

Date: 2026-10-30 · Status: Accepted

## Context

Two independent choices, both with a live trap underneath.

**Test framework.** On the .NET 10 SDK, `dotnet new xunit` still scaffolds xunit v2 — which is
marked deprecated on NuGet, security fixes only — together with a `Microsoft.NET.Test.Sdk` four
minor versions behind. Reaching for `xunit.v3` instead does not just work either: version 4.0.0 and
later enable Microsoft.Testing.Platform v2 by default, while `dotnet test` on the .NET 10 SDK still
defaults to VSTest, and the mismatch fails the run outright:

```
Testing with VSTest target is no longer supported by Microsoft.Testing.Platform on .NET 10 SDK and later.
```

xUnit closed the corresponding issue as Won't Fix on 2026-10-06 — this is intended behaviour, not a
patch to wait for. `TestingPlatformDotnetTestSupport` is the .NET 8/9 mechanism and is not honoured
on .NET 10.

**Broker under test.** `Testcontainers.Mosquitto` 4.16.0 exists and is an official module.
`Testcontainers.EMQX` does not exist; EMQX requires a hand-built `ContainerBuilder` with its own
wait strategy.

## Decision

**Framework:** `xunit.v3` 4.0.2 with the runner selected in `global.json`:

```json
{ "test": { "runner": "Microsoft.Testing.Platform" } }
```

In this mode neither `Microsoft.NET.Test.Sdk` nor `xunit.runner.visualstudio` is referenced. The
test project is not bootstrapped from the in-box template.

**Broker:** EMQX 6.3.2, built with `ContainerBuilder` and an `emqx ctl status` wait strategy — the
same command the compose healthcheck uses.

**Substitutes:** NSubstitute. The database is never mocked; integration tests use a real MariaDB
through Testcontainers.

## Consequences

- Choosing EMQX over the supported Mosquitto module costs a hand-written container definition and
  roughly fifteen seconds of startup per fixture.
- It buys the only thing that matters here: Last Will semantics, session takeover, and session
  expiry are exactly where broker implementations differ, and those behaviours are under test.
  Asserting them against a broker we do not deploy would prove nothing about the system we ship.
- Containers are shared per collection fixture, not created per test class, so the startup cost is
  paid once.
- Asynchronous outcomes are awaited by polling with a timeout. `Thread.Sleep` is banned: it is
  simultaneously too slow and too flaky.
- Test configuration overrides connection strings before the DI container is built, not after.
- CI must provide a Docker daemon for Testcontainers.
