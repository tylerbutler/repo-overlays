# Type Test Updates (Steps 8-9)

Update type test baselines on both `main` and the release branch. Do this **the day after** the release to allow ADO npm feeds to pick up the published packages.

## Step 8: Update Type Test Baselines on Main

```bash
git checkout main
git pull
```

### Reset and regenerate

```bash
pnpm exec flub typetests -g client --reset --normalize --previous
pnpm install --no-frozen-lockfile
```

Then either do a full build or generate type tests only:

```bash
# Option A: Full build
pnpm run build

# Option B: Type tests only (faster)
pnpm run typetests:gen
```

Commit and create a PR targeting `main`.

## Step 9: Update Type Test Baselines on Release Branch

Switch to the release branch and repeat the same process:

```bash
git checkout release/client/<MAJOR>.<MINOR>
git pull
```

```bash
pnpm exec flub typetests -g client --reset --normalize --previous
pnpm install --no-frozen-lockfile
pnpm run typetests:gen
```

Commit and create a PR targeting the release branch.

For the first release of a new minor (X.X0.0), this PR can be combined with the patch version bump PR from Step 7 to avoid running the release branch patch process twice.
