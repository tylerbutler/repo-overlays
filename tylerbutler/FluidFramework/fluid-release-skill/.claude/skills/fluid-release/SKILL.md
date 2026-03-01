---
name: fluid-release
description: Guide the Fluid Framework client release group through minor releases, patch releases, and post-release type test updates. Use when performing a release, preparing for a release, creating release branches, bumping versions, generating changelogs/release notes, or updating type test baselines after a release. Triggers on mentions of release, releasing, release branch, version bump, release notes, changelog generation, patch release, minor release, or release engineering.
---

# Fluid Framework Client Release

Interactive release workflow for the client release group. Runs commands autonomously but pauses before creating PRs, pushing branches, or triggering builds.

## Workflow Selection

Ask the user which phase they need:

| Phase | When to use | Reference |
|-------|-------------|-----------|
| **Minor release prep** | Starting a new minor release from `main` (Steps 1-5) | [minor-release-prep.md](references/minor-release-prep.md) |
| **Release execution** | Running the release build + patch bump (Steps 6-7). Also used for **patch releases** on existing branches. | [release-execution.md](references/release-execution.md) |
| **Type test updates** | Day after release: update baselines on main and release branch (Steps 8-9) | [type-test-updates.md](references/type-test-updates.md) |

For **patch releases**, skip directly to release execution on an existing release branch.

## Key Context

- The repo uses `pnpm` as the package manager
- `flub` is the Fluid build CLI (`pnpm flub ...` or `pnpm exec flub ...`)
- Client minor versions follow the `2.X0.0` pattern (e.g., 2.80.0, 2.90.0, 2.100.0)
- Patches increment within the decade (2.80.0 -> 2.81.0 -> 2.82.0)
- Release branch naming: `release/client/<major>.<minor>` (e.g., `release/client/2.90`)
- The release branch is created from the commit **before** the version bump on `main`
- There is no `lerna.json` in this repo

## Before Starting

Run `pnpm flub release prepare client` to check readiness. Also check for release blockers:
- [GitHub release-blocking issues](https://github.com/microsoft/FluidFramework/labels/release-blocking)
- ADO release-blocking issues (check manually)

## Semi-Autonomous Behavior

**Run automatically:** `policy-check:asserts`, `layerGeneration:gen`, `flub generate releaseNotes`, `flub generate changelog`, `flub bump`, `build:genver`, `flub typetests`, `flub release prepare`

**Pause and confirm before:** creating PRs, pushing branches, running `flub release`, announcing releases

Read the appropriate reference file for the phase the user selects, then guide them through it step by step.
