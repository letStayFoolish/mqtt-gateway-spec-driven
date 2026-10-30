# MG-020 — README and known limitations

Depends on: MG-018

## Goal

Someone who is not the author can run, understand, and operate this project.

## Scope

- What it is, in two sentences, and the architecture in one diagram.
- Prerequisites, and how to run it two ways: everything in compose, or dependencies in compose with
  `dotnet run` on the host.
- Every environment variable, what it does, and its default.
- How migrations are applied, and what to do when they fail.
- How to run the tests, including the Docker requirement for Testcontainers.
- Links to [`spec.md`](docs/spec.md), [`spec-gaps.md`](docs/spec-gaps.md), and [`decisions/`](docs/decisions/).
- **Known limitations**, stated plainly rather than discovered later:
  - One instance per `ClientId`; horizontal scaling needs shared subscriptions (ADR-0005)
  - Database and Redis can disagree after a partial failure; there is no distributed transaction
    (ADR-0004)
  - Deferred acknowledgement means redelivery after a crash can produce duplicate rows (ADR-0003)
  - No API authentication, by decision (spec §2)
  - Statistics have one-minute resolution (ADR-0004)
  - EMQX 6.x is licensed under BSL 1.1 — free for non-production and single-node production use
  - The EF provider is a community fork; the reasoning and the exit path are in ADR-0001

## Acceptance criteria

- [ ] A reader who has never seen the repository can bring the stack up by following it exactly
- [ ] Every configuration key is documented
- [ ] No known limitation is discovered by a reader rather than disclosed by the README

## Gaps closed

GAP-44 (credentials in documentation), and the "production-ready" claim the original assignment
made without disclosing what was missing.
