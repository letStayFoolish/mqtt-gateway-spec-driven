# 01 — Repository skeleton, solution, central packages, CI

Status: ready-for-agent
Depends on: —

## Goal

`git clone` → `dotnet build` → `dotnet test` succeeds on a machine with the .NET 10 SDK and nothing
else. No application behaviour yet.

## Scope

- `.gitignore` for .NET; `bin/` and `obj/` never enter git.
- `global.json`: SDK `10.0.403` with `rollForward: latestFeature`, and
  `"test": { "runner": "Microsoft.Testing.Platform" }`.
- `MqttGateway.slnx` with `src/MqttGateway.Api`, `tests/MqttGateway.UnitTests`,
  `tests/MqttGateway.IntegrationTests`.
- `Directory.Build.props`: `net10.0`, `Nullable enable`, `ImplicitUsings enable`,
  `LangVersion latest`, `EnforceCodeStyleInBuild`, `AnalysisLevel latest-recommended`,
  `TreatWarningsAsErrors` in CI only.
- `Directory.Packages.props` with central package management; every version pinned per spec §3.
- Test projects reference `xunit.v3` 4.0.2 only — **no** `Microsoft.NET.Test.Sdk`, **no**
  `xunit.runner.visualstudio`. Do not scaffold with `dotnet new xunit`.
- `InternalsVisibleTo` for both test projects.
- `.editorconfig`.
- GitHub Actions: restore → build → test on every push and pull request, with a Docker daemon
  available for later integration tests.

## Acceptance criteria

- [ ] `dotnet build` produces zero warnings
- [ ] `dotnet test` runs and reports zero tests without error — proving the runner is wired
- [ ] CI green
- [ ] `git status` clean after a build

## Gaps closed

GAP-40 (deprecated xunit v2 / MTP runner mismatch), GAP-36 (.NET 8 end of support).

## Traps

Omitting the `global.json` test runner block makes `dotnet test` fail with
`Testing with VSTest target is no longer supported…`. `TestingPlatformDotnetTestSupport` does not
work on .NET 10 — it is the .NET 8/9 mechanism.
