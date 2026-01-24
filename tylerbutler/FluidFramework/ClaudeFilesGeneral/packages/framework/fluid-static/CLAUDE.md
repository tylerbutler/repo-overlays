# @fluidframework/fluid-static

Enables consumption of Fluid collaborative data without requiring custom container code. This package provides the simplified, high-level API layer that service-specific clients (like `@fluidframework/azure-client` and `@fluidframework/tinylicious-client`) build upon.

## Purpose

This package abstracts away the complexity of Fluid's container runtime, data stores, and factories. Instead of manually wiring up container code, developers define a `ContainerSchema` and get back an `IFluidContainer` with pre-instantiated collaborative objects.

## Architecture

### Core Components

**IFluidContainer** (`fluidContainer.ts`)
- Main entry point for interacting with collaborative data
- Wraps the underlying `IContainer` and provides a simplified API
- Exposes `initialObjects` (pre-created DDSes/DataObjects), connection state, and dirty state
- Emits events: `connected`, `disconnected`, `saved`, `dirty`, `disposed`
- Note: The base `FluidContainer` class throws on `attach()` - service-specific implementations must override this

**ContainerSchema / TreeContainerSchema** (`types.ts`)
- Declares what collaborative objects exist in a container
- `initialObjects`: Objects created when the container is first created (keyed by string ID)
- `dynamicObjectTypes`: Additional object types that can be created at runtime
- `TreeContainerSchema` is a specialized variant for SharedTree-only containers

**RootDataObject** (`rootDataObject.ts`)
- Internal DataObject that serves as the container's entry point
- Stores handles to all `initialObjects` in a SharedDirectory
- Provides `create()` method for dynamic object creation
- Service clients never interact with this directly

**TreeRootDataObject** (`treeRootDataObject.ts`)
- Alternative root DataObject for tree-based containers
- Extends `TreeDataObject` from aqueduct
- Exposes a single `tree` property as the initial object

**ServiceAudience** (`serviceAudience.ts`)
- Tracks connected users/members in a collaborative session
- Aggregates multiple connections from the same user
- Filters out non-interactive clients (e.g., summarizer)
- Emits `memberAdded`, `memberRemoved`, `membersChanged` events

**CompatibilityMode** (`compatibilityConfiguration.ts`)
- `"1"`: Full interop with 1.x clients
- `"2"`: Only 2.x client interop, enables SharedTree prerequisites (ID compressor)

### Type Utilities

**InitialObjects<T>** - Extracts the type of instantiated objects from a `ContainerSchema`:
```typescript
const schema = { initialObjects: { myMap: SharedMap, myTree: SharedTree } };
// InitialObjects<typeof schema> = { myMap: SharedMap; myTree: ITree }
```

**LoadableObjectKind** - Union type representing either a SharedObject kind or DataObject kind.

## Key Exports

```typescript
// Container creation (used by service clients)
createFluidContainer<T>()           // Creates IFluidContainer from IContainer
createDOProviderContainerRuntimeFactory()  // Creates runtime factory for DataObject-based containers
createTreeContainerRuntimeFactory()        // Creates runtime factory for tree-based containers

// Audience
createServiceAudience<M>()          // Creates service audience from IContainer

// Types
IFluidContainer, IFluidContainerEvents, InitialObjects
ContainerSchema, TreeContainerSchema
IServiceAudience, IMember, IConnection, Myself
CompatibilityMode, ContainerAttachProps
```

## Testing

Tests are in `src/test/` and use mocha. The tests focus on:

- **Type tests** (`fluidContainer.spec.ts`): Compile-time verification that `InitialObjects<T>` correctly infers types from schemas
- **Unit tests** (`utils.spec.ts`):
  - `parseDataObjectsFromSharedObjects`: Separates DDSes from DataObjects in a schema, handles deduplication
  - `isTreeContainerSchema`: Validates that a schema conforms to tree container requirements (single `tree` key with SharedTree)

Run tests:
```bash
pnpm test:mocha
```

## Implementation Notes

- The `FluidContainer.attach()` method intentionally throws - actual attachment logic must be provided by service-specific implementations
- `parseDataObjectsFromSharedObjects` automatically deduplicates types that appear in both `initialObjects` and `dynamicObjectTypes`
- The audience only includes "interactive" clients (human users), filtering out system clients like the summarizer
- `TreeContainerSchema` requires exactly one initial object with key `"tree"` containing a SharedTree
