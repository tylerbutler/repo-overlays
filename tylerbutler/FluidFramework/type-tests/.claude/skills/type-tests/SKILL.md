---
name: type-tests
description: Manage type compatibility tests in the Fluid Framework monorepo. Use when type tests fail, when acknowledging breaking changes, or when preparing type test baselines. Triggers on mentions of type tests, typetests, type validation, type compatibility, backward/forward compat, or validate*Previous.generated.ts.
---

# Type Test Management

Manage the automated type compatibility testing system in the Fluid Framework monorepo.

## Overview

Type tests verify that the current version of a package's types are compatible with the previous released version. This catches accidental breaking changes before they ship.

The system has two phases:
1. **Prepare** (`flub typetests`) — updates `package.json` config and the `-previous` dev dependency
2. **Generate** (`flub generate typetests`) — creates the actual test files

## How It Works

Each package has a dev dependency on its own previous version:

```json
"devDependencies": {
  "@fluidframework/tree-previous": "npm:@fluidframework/tree@2.83.0"
}
```

Generated test files import both `old` (previous) and `current` types and check assignability in both directions:

```typescript
import type * as old from "@fluidframework/tree-previous";
import type * as current from "../../index.js";

// Backward compat: can current substitute for old?
declare type current_as_old_for_FieldSchema =
  requireAssignableTo<TypeOnly<current.FieldSchema>, TypeOnly<old.FieldSchema>>

// Forward compat: can old substitute for current?
declare type old_as_current_for_FieldSchema =
  requireAssignableTo<TypeOnly<old.FieldSchema>, TypeOnly<current.FieldSchema>>
```

## Configuration in package.json

```json
"typeValidation": {
  "broken": {},
  "entrypoint": "public"
}
```

| Field | Purpose |
|-------|---------|
| `entrypoint` | Which API level to test: `"public"`, `"beta"`, `"alpha"`, `"legacy"` |
| `broken` | Map of type names to known breaking changes |
| `disabled` | Set to `true` to skip type tests entirely (rare) |

## Generated Test Files

Located at: `src/test/types/validate<PackageName>Previous.generated.ts`

These files are committed to the repo and should not be manually edited.

## Common Workflows

### Regenerating Type Tests

After a version bump or when tests are out of date:

```bash
cd packages/path/to/package

# Prepare: update -previous dependency and reset broken list
pnpm typetests:prepare
# Equivalent to: flub typetests --dir . --reset --previous --normalize

# Install the new previous version
pnpm install

# Generate the test files
pnpm typetests:gen
# Equivalent to: flub generate typetests --dir . -v
```

### When Type Tests Fail

Type test failures mean the current types are incompatible with the previous version. You have two options:

#### Option A: Fix the incompatibility

If the change was unintentional, revert or adjust the types to restore compatibility.

#### Option B: Acknowledge the breaking change

If the change is intentional, add an entry to `typeValidation.broken` in `package.json`:

```json
"typeValidation": {
  "broken": {
    "ClassName": { "backCompat": false },
    "InterfaceName": { "forwardCompat": false }
  }
}
```

| Key | Meaning |
|-----|---------|
| `backCompat: false` | Current type cannot substitute for old type (removed members, narrowed types) |
| `forwardCompat: false` | Old type cannot substitute for current type (added required members) |

You can set both to `false` if both directions are broken.

After adding broken entries, regenerate:

```bash
pnpm typetests:gen
```

### Special Tags That Affect Generation

| TSDoc Tag | Effect on Type Tests |
|-----------|---------------------|
| `@sealed` | Skips forward compatibility tests (consumers can't extend, so adding members is safe) |
| `@input` | Skips backward compatibility tests (input types can safely narrow) |
| `@system` | Skips generation entirely (internal system types) |

## flub typetests Flags (Prepare Phase)

```bash
flub typetests --dir <path> [flags]
```

| Flag | Purpose |
|------|---------|
| `--reset` | Clear all entries in `typeValidation.broken` |
| `--previous` / `-p` | Set `-previous` dependency to the version before current |
| `--exact <version>` | Set `-previous` dependency to a specific version |
| `--normalize` / `-n` | Clean up config (remove unrecognized fields, add defaults) |
| `--remove` / `-r` | Remove the `-previous` dev dependency |
| `--enable` | Remove `typeValidation.disabled` |
| `--disable` | Set `typeValidation.disabled: true` |

Typical combo: `flub typetests --dir . --reset --previous --normalize`

## flub generate typetests Flags (Generate Phase)

```bash
flub generate typetests --dir <path> [flags]
```

| Flag | Purpose |
|------|---------|
| `--entrypoint <value>` | Override `typeValidation.entrypoint` |
| `--outDir <path>` | Output directory (default: `./src/test/types`) |
| `--outFile <name>` | Output file name pattern |
| `--publicFallback` | Fall back to public entrypoint if specified one missing |
| `-v, --verbose` | Verbose output |

## Type Helpers

The generated tests use three type preprocessors from `@fluidframework/build-tools`:

| Helper | Purpose |
|--------|---------|
| `TypeOnly<T>` | Tests type shape only (default) |
| `MinimalType<T>` | Tests minimal structural compatibility |
| `FullType<T>` | Tests full structural compatibility |

## Full Workflow After a Release

1. `flub typetests --dir . --reset --previous --normalize` — point to new previous version, clear broken list
2. `pnpm install` — fetch the previous version package
3. `flub generate typetests --dir . -v` — generate fresh test files
4. Build and run tests — if failures, either fix types or add `broken` entries
5. Commit the updated `package.json` and generated test files
