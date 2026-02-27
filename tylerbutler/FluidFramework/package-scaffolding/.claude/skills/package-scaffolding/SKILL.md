---
name: package-scaffolding
description: Create new packages in the Fluid Framework monorepo following all conventions. Use when creating a new package, adding a new DDS, or scaffolding a new library. Triggers on mentions of new package, scaffold, create package, add package, or package template.
---

# Package Scaffolding

Guide for creating new packages in the Fluid Framework monorepo with all required conventions.

## Where to Place the Package

| Directory | Scope | Purpose |
|-----------|-------|---------|
| `packages/common/` | `@fluidframework/*` | Shared interfaces, utilities |
| `packages/dds/` | `@fluidframework/*` | Distributed Data Structures |
| `packages/drivers/` | `@fluidframework/*` | Service drivers |
| `packages/framework/` | `@fluidframework/*` | Framework components |
| `packages/loader/` | `@fluidframework/*` | Container loading |
| `packages/runtime/` | `@fluidframework/*` | Runtime components |
| `packages/service-clients/` | `@fluidframework/*` | Service client implementations |
| `packages/utils/` | `@fluidframework/*` | Shared utilities |
| `packages/test/` | `@fluid-internal/*` | Test utilities (not published) |
| `packages/tools/` | `@fluidframework/*` | Developer tools |
| `experimental/` | `@fluid-experimental/*` | Experimental packages |
| `azure/packages/` | `@fluidframework/*` | Azure-specific packages |
| `examples/` | `@fluid-example/*` | Examples (not published) |

All these paths are included in `pnpm-workspace.yaml`. No workspace registration is needed.

## Directory Structure

```
my-package/
├── api-extractor/
│   ├── api-extractor.current.json
│   ├── api-extractor.legacy.json
│   ├── api-extractor-lint-public.esm.json
│   ├── api-extractor-lint-public.cjs.json
│   ├── api-extractor-lint-legacy.esm.json
│   ├── api-extractor-lint-legacy.cjs.json
│   └── api-extractor-lint-bundle.json
├── api-report/                          # Generated, committed
│   └── (created after first build)
├── src/
│   ├── cjs/
│   │   └── package.json
│   ├── test/
│   │   ├── tsconfig.json
│   │   ├── tsconfig.cjs.json
│   │   └── *.spec.ts
│   ├── index.ts
│   └── internal.ts
├── .mocharc.cjs
├── eslint.config.mts
├── package.json
├── test-config.json
├── tsconfig.json
└── tsconfig.cjs.json
```

## Required Files

### package.json

Adapt the version, name, description, and directory path. Match devDependency versions to an existing peer package.

```json
{
  "name": "@fluidframework/my-package",
  "version": "2.90.0",
  "description": "Description of the package",
  "homepage": "https://fluidframework.com",
  "repository": {
    "type": "git",
    "url": "https://github.com/microsoft/FluidFramework.git",
    "directory": "packages/category/my-package"
  },
  "license": "MIT",
  "author": "Microsoft and contributors",
  "sideEffects": false,
  "type": "module",
  "exports": {
    ".": {
      "import": {
        "types": "./lib/public.d.ts",
        "default": "./lib/index.js"
      },
      "require": {
        "types": "./dist/public.d.ts",
        "default": "./dist/index.js"
      }
    },
    "./internal": {
      "import": {
        "types": "./lib/index.d.ts",
        "default": "./lib/index.js"
      },
      "require": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      }
    }
  },
  "main": "lib/index.js",
  "types": "lib/public.d.ts",
  "scripts": {
    "api": "fluid-build . --task api",
    "api-extractor:commonjs": "flub generate entrypoints --outDir ./dist",
    "api-extractor:esnext": "flub generate entrypoints --outDir ./lib --node10TypeCompat",
    "build": "fluid-build . --task build",
    "build:api-reports": "concurrently \"npm:build:api-reports:*\"",
    "build:api-reports:current": "api-extractor run --local --config api-extractor/api-extractor.current.json",
    "build:commonjs": "fluid-build . --task commonjs",
    "build:compile": "fluid-build . --task compile",
    "build:docs": "api-extractor run --local",
    "build:esnext": "tsc --project ./tsconfig.json",
    "build:test": "npm run build:test:esm && npm run build:test:cjs",
    "build:test:cjs": "fluid-tsc commonjs --project ./src/test/tsconfig.cjs.json",
    "build:test:esm": "tsc --project ./src/test/tsconfig.json",
    "check:are-the-types-wrong": "attw --pack .",
    "check:biome": "biome check .",
    "check:exports": "concurrently \"npm:check:exports:*\"",
    "check:exports:esm:public": "api-extractor run --config api-extractor/api-extractor-lint-public.esm.json",
    "check:exports:cjs:public": "api-extractor run --config api-extractor/api-extractor-lint-public.cjs.json",
    "check:exports:bundle-release-tags": "api-extractor run --config api-extractor/api-extractor-lint-bundle.json",
    "clean": "rimraf --glob dist lib \"*.d.ts\" \"**/*.tsbuildinfo\" \"**/*.build.log\" _api-extractor-temp nyc",
    "eslint": "eslint --quiet --format stylish src",
    "eslint:fix": "eslint --quiet --format stylish src --fix --fix-type problem,suggestion,layout",
    "format": "npm run format:biome",
    "format:biome": "biome check . --write",
    "lint": "fluid-build . --task lint",
    "lint:fix": "fluid-build . --task eslint:fix --task format",
    "test": "npm run test:mocha",
    "test:coverage": "c8 npm test",
    "test:mocha": "npm run test:mocha:esm && echo skipping cjs to avoid overhead",
    "test:mocha:cjs": "cross-env FLUID_TEST_MODULE_SYSTEM=CJS mocha",
    "test:mocha:esm": "mocha",
    "tsc": "fluid-tsc commonjs --project ./tsconfig.cjs.json && npm run place:cjs:package-stub",
    "place:cjs:package-stub": "copyfiles -f ./src/cjs/package.json ./dist",
    "typetests:gen": "flub generate typetests --dir . -v",
    "typetests:prepare": "flub typetests --dir . --reset --previous --normalize"
  },
  "c8": {
    "all": true,
    "cache-dir": "nyc/.cache",
    "exclude": ["src/test/**/*.*ts", "lib/test/**/*.*js"],
    "exclude-after-remap": false,
    "include": ["src/**/*.*ts", "lib/**/*.*js"],
    "report-dir": "nyc/report",
    "reporter": ["cobertura", "html", "text"],
    "temp-directory": "nyc/.nyc_output"
  },
  "dependencies": {},
  "devDependencies": {
    "@arethetypeswrong/cli": "^0.18.2",
    "@biomejs/biome": "~1.9.3",
    "@fluid-tools/build-cli": "^0.63.0",
    "@fluidframework/build-common": "^2.0.3",
    "@fluidframework/build-tools": "^0.63.0",
    "@fluidframework/eslint-config-fluid": "workspace:~",
    "@microsoft/api-extractor": "7.52.11",
    "@types/mocha": "^10.0.10",
    "@types/node": "~20.19.30",
    "c8": "^10.1.3",
    "concurrently": "^9.2.1",
    "copyfiles": "^2.4.1",
    "cross-env": "^10.1.0",
    "eslint": "~9.39.1",
    "mocha": "^11.7.5",
    "mocha-multi-reporters": "^1.5.1",
    "rimraf": "^6.1.3",
    "typescript": "~5.4.5"
  },
  "fluidBuild": {
    "tasks": {}
  },
  "typeValidation": {
    "broken": {},
    "entrypoint": "public"
  }
}
```

**Key conventions:**
- Use `workspace:~` for internal Fluid dependencies
- Match the monorepo version (`2.90.0` or current)
- Use `"type": "module"` for ESM-first
- Internal dependencies go in `dependencies`, build tools in `devDependencies`

### tsconfig.json (ESM)

Adjust the `extends` path based on package depth relative to `common/build/build-common/`.

```json
{
  "extends": "../../../common/build/build-common/tsconfig.node16.json",
  "include": ["src/**/*"],
  "exclude": ["src/test/**/*"],
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./lib"
  }
}
```

### tsconfig.cjs.json (CJS)

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist"
  }
}
```

### src/index.ts

```typescript
/*!
 * Copyright (c) Microsoft Corporation and contributors. All rights reserved.
 * Licensed under the MIT License.
 */

export type { IMyInterface } from "./myInterface.js";
export { MyClass } from "./myClass.js";
```

Tag every export with a release tag (`@public`, `@beta`, `@alpha`, `@internal`) in the source file where it's defined.

### src/internal.ts

Re-exports everything from `index.ts` plus any internal-only exports:

```typescript
/*!
 * Copyright (c) Microsoft Corporation and contributors. All rights reserved.
 * Licensed under the MIT License.
 */

export * from "./index.js";
```

### src/cjs/package.json

```json
{
  "type": "commonjs"
}
```

This gets copied to `dist/` during build to mark CJS output.

### eslint.config.mts

Adjust the import path depth to reach `common/build/eslint-config-fluid/`.

```typescript
import type { Linter } from "eslint";
import { strict } from "../../../common/build/eslint-config-fluid/flat.mts";

const config: Linter.Config[] = [
  ...strict,
];

export default config;
```

Available presets: `strict` (core packages), `recommended` (mid-level), `minimalDeprecated` (minimal).

### .mocharc.cjs

```javascript
"use strict";

const testCJS = process.env.FLUID_TEST_MODULE_SYSTEM === "CJS";
const outputFilePrefix = testCJS ? "CJS-" : "";
const suiteName = "@fluidframework/my-package" + (testCJS ? " - CJS" : "");

module.exports = {
  spec: testCJS ? "dist/test/**/*.spec.*js" : "lib/test/**/*.spec.*js",
  recursive: true,
  reporter: "mocha-multi-reporters",
  "reporter-options": [
    `configFile=test-config.json,cmrOutput=xunit+output+${outputFilePrefix}:xunit+suiteName+${suiteName}`,
  ],
  "unhandled-rejections": "strict",
};
```

### test-config.json

```json
{
  "reporterEnabled": "spec, xunit",
  "xunitReporterOptions": {
    "output": "nyc/{id}junit-report.xml",
    "suiteName": "{id}"
  }
}
```

### src/test/tsconfig.json

Adjust the `extends` path depth.

```json
{
  "extends": "../../../../common/build/build-common/tsconfig.test.node16.json",
  "compilerOptions": {
    "rootDir": "./",
    "outDir": "../../lib/test",
    "types": ["mocha", "node"]
  },
  "include": ["./**/*"],
  "references": [
    { "path": "../.." }
  ]
}
```

### API Extractor Configs

#### api-extractor/api-extractor.current.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/api-extractor/v7/api-extractor.schema.json",
  "extends": "<projectFolder>/../../../common/build/build-common/api-extractor-report.esm.current.json",
  "mainEntryPointFilePath": "<projectFolder>/lib/public.d.ts"
}
```

#### api-extractor/api-extractor.legacy.json

Only needed if the package has a legacy API surface. Otherwise omit.

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/api-extractor/v7/api-extractor.schema.json",
  "extends": "<projectFolder>/../../../common/build/build-common/api-extractor-report.esm.legacy.json"
}
```

#### api-extractor/api-extractor-lint-public.esm.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/api-extractor/v7/api-extractor.schema.json",
  "extends": "<projectFolder>/../../../common/build/build-common/api-extractor-lint.entrypoint.json",
  "mainEntryPointFilePath": "<projectFolder>/lib/public.d.ts"
}
```

#### api-extractor/api-extractor-lint-public.cjs.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/api-extractor/v7/api-extractor.schema.json",
  "extends": "<projectFolder>/../../../common/build/build-common/api-extractor-lint.entrypoint.json",
  "mainEntryPointFilePath": "<projectFolder>/dist/public.d.ts"
}
```

#### api-extractor/api-extractor-lint-bundle.json

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/api-extractor/v7/api-extractor.schema.json",
  "extends": "<projectFolder>/../../../common/build/build-common/api-extractor-lint.json",
  "mainEntryPointFilePath": "<projectFolder>/lib/index.d.ts"
}
```

Add additional lint configs for `./legacy`, `./beta`, or `./alpha` paths if the package exposes them.

## Register in layerInfo.json

Add the package to the appropriate group and layer in `layerInfo.json` at the repo root:

```json
"LayerName": {
  "packages": ["@fluidframework/my-package"],
  "deps": ["Core-Interfaces", "Core-Utils"]
}
```

Choose dependencies carefully — they define the allowed dependency graph. Run `flub check layers --info layerInfo.json` to validate.

## Post-Creation Steps

1. Run `pnpm install` to update the workspace lockfile
2. Build the package: `fluid-build packages/category/my-package --task build`
3. Generate API reports: `cd packages/category/my-package && npm run build:api-reports`
4. Commit the generated `api-report/` files
5. Run `flub check policy` to verify repo policy compliance
6. Run `flub check layers --info layerInfo.json` to verify layer dependencies

## Biome Formatting

Packages inherit from the root `biome.jsonc` automatically. No per-package config needed unless you need overrides (rare).

Default formatting: tabs, 95-char line width, double quotes, semicolons always, trailing commas.

## Checklist

- [ ] Directory created in correct location
- [ ] `package.json` with correct name, version, exports, scripts, and dependencies
- [ ] `tsconfig.json` extending `tsconfig.node16.json`
- [ ] `tsconfig.cjs.json` extending local `tsconfig.json`
- [ ] `src/index.ts` with public exports
- [ ] `src/internal.ts` re-exporting index
- [ ] `src/cjs/package.json` with `"type": "commonjs"`
- [ ] `eslint.config.mts` extending `strict`
- [ ] `.mocharc.cjs` with correct package name
- [ ] `test-config.json` for test reporters
- [ ] `src/test/tsconfig.json` for test compilation
- [ ] `api-extractor/` configs (current + lint variants)
- [ ] Package registered in `layerInfo.json`
- [ ] `pnpm install` run to update lockfile
- [ ] First build succeeds
- [ ] `api-report/` generated and committed
- [ ] `flub check policy` passes
- [ ] `flub check layers` passes
