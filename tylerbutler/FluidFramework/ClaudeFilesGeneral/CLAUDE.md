# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Fluid Framework is a library for building distributed, real-time collaborative web applications. The repo contains multiple pnpm workspaces (release groups) that are versioned independently:

- **client** (root) - Main client packages (`packages/`, `azure/`, `examples/`, `experimental/`)
- **server/routerlicious** - Reference Fluid ordering service
- **server/gitrest**, **server/historian** - Server auxiliary services
- **build-tools** - Build infrastructure and CLIs (`flub`, `fluid-build`)

## Build Commands

```bash
# Install dependencies (required first)
pnpm install

# Full build with lint and docs
npm run build

# Fast incremental build (recommended for development)
npm run build:fast

# Build a specific package (from package directory)
cd packages/dds/tree && pnpm build

# Build specific path only
npm run build:fast -- packages/dds/tree

# TypeScript compilation only
npm run tsc:fast

# Format code (biome)
npm run format
npm run format:changed:main  # Only changed files

# Lint
npm run eslint

# Policy checks
npm run policy-check
npm run policy-check:fix
```

## Testing

```bash
# Run all tests (mocha + jest)
npm run test

# Run mocha tests only
npm run test:mocha

# Run jest tests only
npm run test:jest

# Run tests for a single package
cd packages/dds/tree && pnpm test:mocha

# Run a single test file with mocha
ts-mocha path/to/test.spec.ts

# Use .only or .skip in test files to filter tests
```

**Test collateral**: Some tests require the FluidFrameworkTestData submodule:
```bash
git lfs install
git submodule init && git submodule update
```

**VSCode debugging**: Use "Debug Current Mocha Test (auto build)" launch config (F5 from a `.spec.ts` file).

## Architecture

### Package Categories

| Directory | Namespace | Description |
|-----------|-----------|-------------|
| `packages/common/` | `@fluidframework/` | Core interfaces and utilities |
| `packages/dds/` | `@fluidframework/` | Distributed data structures (SharedTree, Map, Sequence) |
| `packages/drivers/` | `@fluidframework/` | Service drivers (ODSP, Routerlicious, Tinylicious) |
| `packages/loader/` | `@fluidframework/` | Container loading infrastructure |
| `packages/runtime/` | `@fluidframework/` | Container and datastore runtime |
| `packages/framework/` | `@fluidframework/` | High-level framework (fluid-static, Aqueduct) |
| `packages/service-clients/` | `@fluidframework/` | Service client libraries (azure-client, tinylicious-client) |
| `experimental/` | `@fluid-experimental/` | Experimental features |
| `examples/` | `@fluid-example/` | Example applications (not published) |

### Layer Dependencies

Dependencies between packages are enforced via `layerInfo.json`. Run `npm run layer-check` to verify. Key layers (bottom to top):
1. **Definitions** - Core interfaces, protocol definitions, driver definitions
2. **Utils** - Common utilities, telemetry utilities
3. **Driver** - Service-specific drivers
4. **Loader** - Container loading
5. **Runtime** - DDSs and runtime infrastructure
6. **Framework** - High-level APIs and service clients

### Build System

- **fluid-build**: Custom task scheduler that handles incremental builds with dependency tracking
- **flub**: CLI for release management, policy checks, and code generation
- Task definitions: `fluidBuild.config.cjs` (repo-wide) and `package.json` `fluidBuild.tasks` (per-package)

### Dual ESM/CJS Builds

Most packages produce both ESM (`lib/`) and CommonJS (`dist/`) outputs:
- ESM: `tsc --project tsconfig.json` → `lib/`
- CJS: `fluid-tsc commonjs` → `dist/`
- API reports generated via `api-extractor` for both

## Key Conventions

### Asserts

When writing asserts from `@fluidframework/core-utils`, use string literal messages (not hex codes) for new asserts.

### Formatting

- Biome for formatting (tabs, 95 char line width)
- Run `npm run format` or configure VSCode for format-on-save

### API Exports

Packages use tiered exports (`/alpha`, `/beta`, `/legacy`, `/internal`):
```typescript
import { stableApi } from "@fluidframework/tree";
import { betaApi } from "@fluidframework/tree/beta";
import { alphaApi } from "@fluidframework/tree/alpha";
```

### Changesets

Use `pnpm changeset` to create changesets for PRs that modify published packages.

## Troubleshooting

- **Build failures with no changes**: `pnpm clean` then rebuild
- **Ghost tests or stale artifacts**: `git clean -Xdf`
- **Debugging slow/hanging**: `git clean -xdf` (removes untracked files too)
- **Submodule issues**: `git submodule update --init --recursive`
