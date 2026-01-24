# fluid-framework

The main public entry point for Fluid Framework client applications. This package re-exports APIs from multiple Fluid packages, providing a unified surface for consumers.

## Purpose

This is a **facade package** - it contains no implementation code, only re-exports from dependency packages. Its role is to:

1. Simplify dependency management for consumers (one import instead of many)
2. Control the public API surface across multiple internal packages
3. Provide tiered API access (`/alpha`, `/beta`, `/legacy`)

## Architecture

The package consists of a single `src/index.ts` file that re-exports from:

| Source Package | What's Exported |
|----------------|-----------------|
| `@fluidframework/tree` | SharedTree, schema types, tree APIs (bulk of exports) |
| `@fluidframework/fluid-static` | IFluidContainer, ContainerSchema, audience types |
| `@fluidframework/core-interfaces` | IFluidHandle, event system, FluidObject |
| `@fluidframework/container-definitions` | AttachState, ICriticalContainerError |
| `@fluidframework/container-loader` | ConnectionState |
| `@fluidframework/map` | SharedMap, SharedDirectory (legacy) |
| `@fluidframework/sequence` | SharedString (legacy) |
| `@fluidframework/shared-object-base` | SharedObjectKind, ISharedObject |

### Export Tiers

```typescript
import { SharedTree } from "fluid-framework";           // Public (stable)
import { ... } from "fluid-framework/beta";             // Beta APIs
import { ... } from "fluid-framework/alpha";            // Alpha APIs (configuredSharedTree)
import { SharedMap } from "fluid-framework/legacy";     // Legacy DDSes
```

### Custom Re-exports

The package wraps some imports to adjust their types:

- **`SharedTree`**: Re-exported with `SharedObjectKind<ITree>` type (removing `@alpha` `ISharedObjectKind` interface)
- **`configuredSharedTree`**: Alpha function for non-default SharedTree configuration (debugging, testing, performance tuning)

## Key Exports by Category

### Container and Connection
- `IFluidContainer`, `ContainerSchema`, `ContainerAttachProps`
- `AttachState`, `ConnectionState`
- `IServiceAudience`, `IMember`, `IConnection`

### Data Structures (DDSes)
- **`SharedTree`** (recommended) - Hierarchical collaborative data
- **`SharedMap`** (legacy) - Key/value store
- **`SharedDirectory`** (legacy) - Hierarchical key/value store
- **`SharedString`** (legacy) - Collaborative text

### Tree Schema System
- `SchemaFactory` - Creates tree node schemas
- `TreeViewConfiguration` - Configures tree views
- `TreeNode`, `TreeArrayNode`, `TreeMapNode` - Base node types
- Field types: `FieldKind`, `FieldSchema`, `FieldProps`

### Event System
- `Listenable`, `Listeners`, `Off` - Event subscription
- `IEventProvider`, `IEvent` - Legacy event interfaces

## Testing

This package has **no tests** - it's purely re-exports. Testing coverage comes from:
1. The source packages being re-exported
2. API report validation (`api-extractor`)
3. Type validation across versions

The `api-report/` directory contains generated API surface documentation for each tier.

## Important Notes

- **Type validation disabled**: `typeValidation.disabled: true` in package.json
- **Event API conflicts**: Tree package exports deprecated event APIs that conflict with core-interfaces; the named exports from core-interfaces take precedence
- **No `/internal` entrypoint**: Unlike source packages, this facade has no internal API tier
