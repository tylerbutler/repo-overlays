# Release Execution (Steps 6-7)

Run the release build and bump the release branch to the next patch version. This applies to both minor (X.X0.0) and patch releases.

## Step 6: Run the Release Build

### Switch to the release branch

```bash
git checkout release/client/<MAJOR>.<MINOR>
git pull
pnpm install
```

### Run the release

```bash
pnpm flub release -g client -t patch
```

**Pause and confirm** before running this command. It will:
- Run checks and prompt for confirmation
- Instruct you to queue a release build in ADO (choosing the "release" option)

Follow the tool's interactive prompts. The user will need to queue the ADO build manually.

### Verify the release

After the build completes, confirm:
- Listed in [GitHub releases](https://github.com/microsoft/FluidFramework/releases)
- Published on [npm](https://www.npmjs.com/search?q=%40fluidframework)

Once confirmed, remind the user to announce the release in the Fluid Framework "General" Teams channel with a link to the GitHub release.

## Step 7: Bump Release Branch to Next Patch

Wait for the release tag to be added to the repo, then either:

### Option A: Use flub release again

```bash
pnpm flub release -g client -t patch
```

This should detect the release and bump the version automatically.

### Option B: Manual bump

```bash
pnpm exec flub bump client --bumpType patch
```

Create a PR targeting the release branch with these changes.

For the first release of a new minor (X.X0.0), this PR can optionally be combined with the type test baseline update from [type-test-updates.md](type-test-updates.md) Step 9.

After the release, proceed to [type test updates](type-test-updates.md) **the next day**.
