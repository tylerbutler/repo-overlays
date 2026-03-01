# Minor Release Prep (Steps 1-5)

Prepare the `main` branch and create the release branch for a new minor release.

## Overview

Open four PRs (can be opened in parallel, but merge order matters):
1. Tag untagged asserts
2. Update compatibility generation
3. Generate release notes and changelogs
4. Bump to next version (**must merge last**)

Then create the release branch from the commit before the version bump.

## Step 1: Tag Untagged Asserts

```bash
pnpm run policy-check:asserts
```

If there are changes, commit them and create a PR. This PR must merge before the version bump PR.

Timing note: do this close to release to minimize untagged asserts being merged afterward.

## Step 2: Update Compatibility Generation

```bash
pnpm run -r layerGeneration:gen
```

This often produces no changes. If there are changes, commit and create a PR. Must merge before the version bump PR.

This generates changes only if 33+ days have passed since the last update for a given package (tracked in `fluidCompatMetadata` in package.json).

## Step 3: Generate Release Notes and Changelogs

### Determine the version being released

Check the current version in the root `package.json`. This is the version being released (the bump hasn't happened yet at this point). Ask the user to confirm the version.

### Generate release notes

```bash
pnpm flub generate releaseNotes -g client -t minor --outFile RELEASE_NOTES/<VERSION>.md
```

**Commit** the release notes before proceeding (the next command deletes changesets).

### Generate per-package changelogs

```bash
pnpm flub generate changelog -g client
```

Commit and create a PR. Must merge before the version bump PR.

### If changeset edits are needed after generation

If feedback requires changeset wording changes:
1. Make changeset edits in a **separate PR**, merge it
2. Regenerate release notes and changelogs
3. This ensures changeset changes have a commit in main (since changesets are deleted during changelog generation)

## Step 4: Bump Main to Next Version

### Determine the next version

Ask the user what the next version should be. Client minor versions use the `2.X0.0` pattern:
- After 2.80.0, the next minor is 2.90.0
- After 2.90.0, the next minor is 2.100.0

### Bump versions

```bash
pnpm flub bump client --exact <NEXT_VERSION> --no-commit
```

### Generate version files

```bash
pnpm -r run build:genver
```

Commit and create a PR. **This PR must merge LAST.**

## Step 5: Create the Release Branch

### Pre-checks
- Verify all four PRs are merged
- Check again for release-blocking bugs on [GitHub](https://github.com/microsoft/FluidFramework/labels/release-blocking) and ADO

### Find the correct commit

The release branch is created from the commit **immediately before** the version bump commit. Use:

```bash
git log --oneline -10
```

Identify the version bump commit and use the commit before it.

### Create the branch

Branch name format: `release/client/<major>.<minor>` (e.g., `release/client/2.90`)

```bash
git checkout -b release/client/<MAJOR>.<MINOR> <COMMIT_BEFORE_BUMP>
git push --set-upstream origin release/client/<MAJOR>.<MINOR>
```

**Pause and confirm** before pushing. The user may not have permissions to create release branches.

After branch creation, proceed to [release execution](release-execution.md).
