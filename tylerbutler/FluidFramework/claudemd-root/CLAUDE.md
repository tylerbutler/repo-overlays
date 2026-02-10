# FluidFramework

Large TypeScript monorepo for the [Fluid Framework](https://fluidframework.com), a library for building distributed, real-time collaborative applications.

## Repository Structure

- **`packages/`** — Core published packages (dds, drivers, framework, loader, runtime, utils, tools, test, common, service-clients)
- **`azure/packages/`** — Azure-specific packages
- **`examples/`** — Example applications
- **`experimental/`** — Experimental packages
- **`build-tools/`** — Internal build tooling (separate pnpm workspace with its own lockfile)
- **`docs/`** — Documentation site
- **`tools/`** — Repository-level tooling (pipelines, markdown-magic)

## Package Management

- Uses **pnpm** (v10). Always use `pnpm`, never npm or yarn.
- Two workspaces: root (`pnpm-workspace.yaml`) for the "client" release group, and `build-tools/` for internal tooling.
- Node.js >=20.19.0 required (see `.nvmrc`).

## Build System

- **`fluid-build`** — Custom build orchestrator. Wraps tsc, eslint, webpack, etc. with dependency-aware task scheduling.
- **`flub`** (aka `@fluid-tools/build-cli`) — CLI for repo-level operations: changesets, policy checks, layer checks, release management.

### Common Commands

```bash
pnpm build              # Full build (all tasks)
pnpm build:fast         # Build with worker parallelism
pnpm tsc                # TypeScript compilation only
pnpm eslint             # Lint all packages
pnpm format             # Format with Biome
pnpm checks             # All checks (format, policy, layers, syncpack, versions)
pnpm policy-check       # Check package.json conventions, headers, etc.
pnpm policy-check:fix   # Auto-fix policy violations
pnpm layer-check        # Validate dependency layering
pnpm clean              # Clean all build outputs
```

### Building a Single Package

Run `fluid-build` from the package directory, or target it:
```bash
cd packages/dds/tree && pnpm build
```

## Formatting and Linting

- **Biome** (v2.3) for formatting. Config in root `biome.jsonc`.
  - Indent: tabs. Line width: 95. Semicolons: always. Trailing commas: all.
  - `package.json` files use line width 1 (one key per line).
  - Some packages override settings (e.g., `experimental/dds/tree` uses line width 120, single quotes).
- **ESLint** (v9, flat config) for linting. Shared config in `common/build/eslint-config-fluid`.
- Run `npx biome check --write .` to format, `pnpm eslint:fix` to auto-fix lint errors.

## Testing

- **Mocha** — Primary test runner for Node.js unit tests.
- **Jest** — Used for browser/DOM tests (requires puppeteer).
- Test commands: `pnpm test:mocha`, `pnpm test:jest`, `pnpm test` (runs all).
- Coverage via `c8`.

## Versioning and Changesets

- Uses **Changesets** with fixed versioning — all packages in the client release group version together.
- Fixed group includes: `@fluid-example/*`, `@fluid-experimental/*`, `@fluid-internal/*`, `@fluid-private/*`, `@fluid-tools/*`, `@fluidframework/*`, `fluid-framework`.
- Create changesets via `pnpm changeset` (wraps `flub changeset add --releaseGroup client`).
- Changeset descriptions are linted with **Vale** (`vale .changeset --glob='*-*-*.md'`).

## Policy and Layer Checks

- `flub check policy` validates package.json conventions, copyright headers, and other repo standards. Run `flub check policy --listHandlers` to see all checks.
- `flub check layers --info layerInfo.json` validates that package dependencies respect the layering architecture.
- Both are part of the `checks` build task.

## API Reports

- Uses **API Extractor** (`@microsoft/api-extractor`) to generate and track public API surfaces.
- API report files (`*.api.md`) are committed and reviewed in PRs.
- Regenerate with `pnpm build:api` in the affected package.

## Key Conventions

- Commit messages follow **conventional commits**: `type(scope): description`
- All TypeScript. Dual ESM/CJS builds for published packages.
- Generated files (e.g., `packageVersion.ts`, `*.generated.ts`, `assertionShortCodesMap.ts`) should not be hand-edited.
- Lock files (`pnpm-lock.yaml`) should not be hand-edited.
