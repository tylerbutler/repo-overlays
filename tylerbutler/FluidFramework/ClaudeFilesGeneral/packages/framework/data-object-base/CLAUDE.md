# CLAUDE.md - @fluid-experimental/data-object-base

## Purpose

Experimental package providing base infrastructure for synchronously and lazily loaded Fluid data objects. It simplifies container runtime initialization by wrapping the boilerplate needed to set up a Fluid container with data stores.

**Note:** This package is experimental. For production use, prefer `@fluidframework/aqueduct`.

## Architecture

### RuntimeFactory

The sole export is `RuntimeFactory`, a class that extends `RuntimeFactoryHelper` from `@fluidframework/runtime-utils`. It handles:

1. **Data store registry creation** - Automatically builds a registry from provided factories
2. **Container runtime initialization** - Calls `loadContainerRuntime` with proper configuration
3. **Default data store creation** - Creates and aliases the default data store on first container instantiation

### Key Components

```
RuntimeFactory
├── constructor(props: RuntimeFactoryProps)
│   ├── defaultStoreFactory: IFluidDataStoreFactory  // Required default factory
│   ├── storeFactories: IFluidDataStoreFactory[]     // Additional factories
│   └── provideEntryPoint: (runtime) => Promise<FluidObject>  // Entry point provider
├── instantiateFirstTime(runtime)  // Creates default data store with empty alias
└── preInitialize(context, existing)  // Loads container runtime with registry
```

### Data Flow

1. `RuntimeFactory` is created with a default data store factory and optional additional factories
2. On first container instantiation, `instantiateFirstTime` creates the default data store
3. The `provideEntryPoint` callback is invoked to expose the container's API to consumers

## Main Exports

| Export | Type | Description |
|--------|------|-------------|
| `RuntimeFactory` | Class | Container runtime factory for data object scenarios |
| `RuntimeFactoryProps` | Interface | Configuration for `RuntimeFactory` constructor |

## Usage Pattern

```typescript
import { RuntimeFactory } from "@fluid-experimental/data-object-base";

const runtimeFactory = new RuntimeFactory({
    defaultStoreFactory: myDataStoreFactory,
    storeFactories: [myDataStoreFactory, otherFactory],
    provideEntryPoint: async (runtime) => {
        const dataStore = await runtime.getAliasedDataStore("");
        return dataStore.entryPoint.get();
    },
});
```

## Testing

This package has no tests. It is a thin wrapper around runtime utilities. Testing is done through integration tests in dependent packages.

## Implementation Details

- **Default store ID**: Uses empty string `""` as the alias for the default data store
- **Registry construction**: Ensures the default factory is always included in the registry, even if not explicitly listed in `storeFactories`
- **API visibility**: All exports are marked `@internal`, indicating they are not part of the public API surface
