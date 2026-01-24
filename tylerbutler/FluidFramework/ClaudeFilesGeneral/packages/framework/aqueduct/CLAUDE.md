# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Package Overview

`@fluidframework/aqueduct` provides base classes for building Fluid objects (data stores) and containers. It's a thin abstraction layer over lower-level Fluid Framework interfaces.

## Architecture

### Class Hierarchy

```
PureDataObject (base - minimal functionality)
├── DataObject (adds SharedDirectory root)
└── TreeDataObject (adds SharedTree root)
```

**PureDataObject** (`src/data-objects/pureDataObject.ts`):
- Bare-bones base providing lifecycle hooks and basic Fluid interfaces
- Lifecycle methods: `preInitialize()` → `initializingFirstTime(props)` / `initializingFromExisting()` → `hasInitialized()`
- Exposes `runtime`, `context`, and `providers` to subclasses

**DataObject** (`src/data-objects/dataObject.ts`):
- Extends PureDataObject with a `root` SharedDirectory
- Most common base class for new data objects

**TreeDataObject** (`src/data-objects/treeDataObject.ts`):
- Extends PureDataObject with a `tree` (SharedTree) instead of SharedDirectory
- For tree-based data models using `@fluidframework/tree`

### Factory Classes

```
PureDataObjectFactory
├── DataObjectFactory (auto-registers SharedDirectory/SharedMap)
└── TreeDataObjectFactory (auto-registers SharedTree)
```

Factories define:
- Data object type name
- Constructor reference
- Shared object factories (DDSes used by the data object)
- Optional provider requirements

### Container Runtime Factories

**BaseContainerRuntimeFactory**: Base for container-level setup

**ContainerRuntimeFactoryWithDefaultDataStore**: Creates containers with a default data object accessible at empty URL. Exposes `fluidExport` for module entry points.

## API Export Tiers

```typescript
import { ... } from "@fluidframework/aqueduct";         // public (stable)
import { ... } from "@fluidframework/aqueduct/legacy";  // legacy APIs
import { ... } from "@fluidframework/aqueduct/internal"; // internal APIs
```

## Key Patterns

### Creating a Data Object

```typescript
class MyDataObject extends DataObject {
  protected async initializingFirstTime(): Promise<void> {
    // Called once on first creation - set up initial state
    const counter = SharedCounter.create(this.runtime);
    this.root.set("counter", counter.handle);
  }

  protected async hasInitialized(): Promise<void> {
    // Called after every initialization - set up runtime state
    this._counter = await this.root.get("counter").get();
  }
}
```

### Creating a Factory

```typescript
export const MyFactory = new DataObjectFactory({
  type: "my-data-object",
  ctor: MyDataObject,
  sharedObjects: [SharedCounter.getFactory()],
});
```

### Container Export

```typescript
export const fluidExport = new ContainerRuntimeFactoryWithDefaultDataStore({
  defaultFactory: MyFactory,
  registryEntries: [MyFactory.registryEntry],
});
```
