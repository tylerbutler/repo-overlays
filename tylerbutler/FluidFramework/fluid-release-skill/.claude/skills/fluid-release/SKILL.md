---
name: fluid-release
description: Guide the Fluid Framework client release group through minor releases, patch releases, and post-release type test updates. Use when performing a release, preparing for a release, creating release branches, bumping versions, generating changelogs/release notes, or updating type test baselines after a release. Triggers on mentions of release, releasing, release branch, version bump, release notes, changelog generation, patch release, minor release, or release engineering.
---

# Fluid Framework Client Release

Release workflow for the client release group. Supports two modes: **interactive** (default) and **autonomous**.

## Mode Selection

At the start of every release session, ask the user:

> Would you like to run in **interactive** or **autonomous** mode?

### Interactive Mode (default)

Run commands autonomously but pause before creating PRs, pushing branches, or triggering builds. Ask for version confirmation at key points. This is the current behavior.

### Autonomous Mode

Run the entire selected phase end-to-end without pausing. Requirements:

- **Version info upfront:** The user must provide all version numbers before starting (current release version and/or next version, depending on phase). Do not prompt for versions mid-flow.
- **No confirmation pauses:** Create PRs, push branches, and run `flub release` without asking. Include clear commit messages and PR descriptions.
- **Phase-scoped execution:** Each phase runs to completion, then reports what the user needs to do next (e.g., "queue the ADO build" or "wait for npm feeds, then re-invoke for type test updates").

If the user provides version info in their initial message (e.g., "release 2.90.0 autonomously"), skip the version questions entirely.

## Workflow Selection

Ask the user which phase they need (or detect from context in autonomous mode):

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

Run `pnpm flub release prepare client` to check readiness. Then check for release blockers:

```bash
gh issue list --repo microsoft/FluidFramework --label release-blocking --state open
gh pr list --repo microsoft/FluidFramework --label release-blocking --state open
```

If either command returns results, **stop and report the blockers to the user**. In autonomous mode, do not proceed past this check if blockers exist. Also remind the user to check ADO for release-blocking issues (cannot be queried via CLI).

## Behavior by Mode

### Commands (both modes)

Run these autonomously in both modes: `policy-check:asserts`, `layerGeneration:gen`, `flub generate releaseNotes`, `flub generate changelog`, `flub bump`, `build:genver`, `flub typetests`, `flub release prepare`

### Checkpoints

| Action | Interactive | Autonomous |
|--------|------------|------------|
| Creating PRs | Pause and confirm | Create automatically with descriptive titles/bodies |
| Pushing branches | Pause and confirm | Push automatically |
| Running `flub release` | Pause and confirm | Run automatically |
| Version determination | Ask user to confirm | Use version provided upfront |
| Announcing releases | Remind user | Remind user (never auto-announce) |
| ADO build queuing | Instruct user | Instruct user (cannot be automated) |

### Autonomous Mode: Phase Completion Reports

At the end of each autonomous phase, provide a summary:

1. **What was done** — list all PRs created, branches pushed, commands run
2. **What to do next** — specific manual steps needed (e.g., queue ADO build, merge PRs in order)
3. **When to continue** — timing guidance for the next phase (e.g., "after PRs merge" or "tomorrow, after npm feeds update")

Read the appropriate reference file for the phase the user selects, then guide them through it step by step.
