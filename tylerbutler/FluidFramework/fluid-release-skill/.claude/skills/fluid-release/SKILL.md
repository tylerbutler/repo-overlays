---
name: fluid-release
description: Guide the Fluid Framework client release group through minor releases, patch releases, and post-release type test updates. Use when performing a release, preparing for a release, creating release branches, bumping versions, generating changelogs/release notes, or updating type test baselines after a release. Also use when the user says generic things like "do the release", "finish the release", "what's the next release", or "release status" — in autonomous mode the agent auto-detects what needs to be done from the schedule and repo state. Triggers on mentions of release, releasing, release branch, version bump, release notes, changelog generation, patch release, minor release, or release engineering.
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

- **Version info upfront:** The user must provide all version numbers before starting (current release version and/or next version, depending on phase). Do not prompt for versions mid-flow. If the user doesn't provide versions, detect them from the schedule and repo state (see below).
- **No confirmation pauses:** Create PRs, push branches, and run `flub release` without asking. Include clear commit messages and PR descriptions.
- **Phase-scoped execution:** Each phase runs to completion, then reports what the user needs to do next (e.g., "queue the ADO build" or "wait for npm feeds, then re-invoke for type test updates").

If the user provides version info in their initial message (e.g., "release 2.90.0 autonomously"), skip the version questions entirely.

#### Auto-detecting release state (autonomous mode)

When the user gives a generic request like "do the release" or "finish the release" without specifying a version or phase, auto-detect from the release schedule and repo state. Read the [release schedule](references/release-schedule.md) and run the detection steps below.

**Step 1: Identify the most recent release.**

```bash
# Get the latest client release tag
git tag -l 'client_v2.*' --sort=-version:refname | head -1
```

**Step 2: Identify the next scheduled release.**

Compare today's date against the schedule. The next release is the earliest scheduled entry whose proposed date is >= today and whose version is greater than the most recently released version. Also check if a release is _overdue_ (proposed date < today but no tag exists).

**Step 3: Check if a release is in progress.**

```bash
# Check for release-prep branches for the next version
git ls-remote --heads upstream 'release-prep/<NEXT_VERSION>/*'
# Check for the release branch
git ls-remote --heads upstream 'release/client/<NEXT_MAJOR>.<NEXT_MINOR>'
# Check for a release tag
git tag -l 'client_v<NEXT_VERSION>'
# Check for open PRs
gh pr list --repo microsoft/FluidFramework --search "release-prep/<NEXT_VERSION>" --state all
```

**Step 4: Determine the phase and act.**

| State | Action |
|-------|--------|
| No release-prep branches, no release branch | Start **minor release prep** (Steps 1-5) |
| Release-prep branches/PRs exist, some not merged | Resume **minor release prep** from where it left off |
| Release branch exists, no release tag | Start **release execution** (Steps 6-7) |
| Release tag exists, no patch bump PR | Resume **release execution** — do the patch bump (Step 7) |
| Release tag exists, patch bump done, no type test PRs | Start **type test updates** (Steps 8-9) |
| All phases complete | Report that the release is fully done and show the next scheduled release |

Present the detected state and chosen action to the user before proceeding. Example:

> **Detected state:** 2.91.0 is scheduled for 03/16/26. No release-prep branches found. The most recent release is 2.90.0.
> **Action:** Starting minor release prep for 2.91.0 (next version on main: 2.92.0).

## Release Schedule

The release schedule is in [references/release-schedule.md](references/release-schedule.md). It contains proposed dates, release versions, and the corresponding "main" version after each release. Use this to determine version numbers and timing in autonomous mode.

## Workflow Selection

Ask the user which phase they need (or auto-detect in autonomous mode — see above):

| Phase | When to use | Reference |
|-------|-------------|-----------|
| **Minor release prep** | Starting a new minor release from `main` (Steps 1-5) | [minor-release-prep.md](references/minor-release-prep.md) |
| **Release execution** | Running the release build + patch bump (Steps 6-7). Also used for **patch releases** on existing branches. | [release-execution.md](references/release-execution.md) |
| **Type test updates** | Day after release: update baselines on main and release branch (Steps 8-9) | [type-test-updates.md](references/type-test-updates.md) |

For **patch releases**, skip directly to release execution on an existing release branch.

## Key Context

- The repo uses `pnpm` as the package manager
- `flub` is the Fluid build CLI (`pnpm flub ...` or `pnpm exec flub ...`)
- **Version scheme**: The version numbering is not a simple incrementing pattern (it is NOT always multiples of 10). When suggesting a next version, default to incrementing the minor version by 1 (e.g., 2.90.0 -> 2.91.0). Trust the version the user provides unless it is more than 7-8 minor versions away from the current version (which likely indicates an error).
- Release branch naming: `release/client/<major>.<minor>` (e.g., `release/client/2.90`)
- The release branch is created from the commit **before** the version bump on `main`
- There is no `lerna.json` in this repo
- **Git remote preference**: When pushing branches, prefer pushing to `upstream` if one is configured for the repo. Check with `git remote -v` if unsure. Only fall back to `origin` if no `upstream` remote exists.
- **Working branch naming**: Do NOT use the `release/` prefix for working branches because `release/` is protected on upstream. Use the standard naming convention below — these branches double as progress markers.

### Working Branch Convention

Working branches follow a numbered naming scheme under `release-prep/<VERSION>/`:

| Step | Branch name |
|------|-------------|
| 1 | `release-prep/<VERSION>/1-tag-asserts` |
| 2 | `release-prep/<VERSION>/2-compat-gen` |
| 3 | `release-prep/<VERSION>/3-release-notes` |
| 4 | `release-prep/<VERSION>/4-bump-<NEXT_VERSION>` |

Example for releasing 2.90.0 with next version 2.91.0:
`release-prep/2.90.0/1-tag-asserts`, `release-prep/2.90.0/4-bump-2.91.0`

### Detecting Prior Progress

Before starting any phase, check for existing progress by looking at branches and PRs:

```bash
# Check for existing release-prep branches on upstream
git ls-remote --heads upstream 'release-prep/<VERSION>/*'
# Check for open release-prep PRs
gh pr list --repo microsoft/FluidFramework --search "release-prep/<VERSION>" --state all
```

If branches or PRs already exist, skip completed steps and resume from where the process left off.

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

### PR Conventions

Use the `build:` conventional commit prefix for all release PR titles (e.g., `build: tag untagged asserts for 2.90.0 release`).

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
