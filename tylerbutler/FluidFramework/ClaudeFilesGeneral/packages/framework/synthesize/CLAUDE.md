# @fluidframework/synthesize

A lightweight IoC (Inversion of Control) container for synthesizing scope objects in Fluid Framework applications.

## Purpose

This package provides dependency injection capabilities, allowing components to declare required and optional dependencies that are resolved at runtime. It implements the `IProvide*` pattern used throughout Fluid Framework, where interfaces expose themselves via a property matching their name (e.g., `IFluidLoadable` exposes `get IFluidLoadable()`).

## Key Concepts

### FluidObject Provider Pattern

Fluid Framework uses a convention where interfaces "provide" themselves:

```typescript
interface IProvideFoo {
  readonly IFoo: IFoo;
}
interface IFoo extends IProvideFoo {
  // actual interface members
}
```

This enables type-safe dependency resolution using string keys that match interface names.

### Provider Types

Providers can be registered in four forms (`FluidObjectProvider<T>`):

1. **Direct value**: `dc.register(IFoo, fooInstance)`
2. **Promise**: `dc.register(IFoo, Promise.resolve(fooInstance))`
3. **Factory function**: `dc.register(IFoo, () => fooInstance)`
4. **Async factory**: `dc.register(IFoo, async (deps) => fooInstance)`

Factory functions receive the `IFluidDependencySynthesizer` to resolve their own dependencies.

## Main Exports

### `DependencyContainer<TMap>`

The core class implementing `IFluidDependencySynthesizer`.

```typescript
// Create container (optionally with parent containers)
const dc = new DependencyContainer<FluidObject<IFoo & IBar>>(parentContainer);

// Register providers
dc.register(IFoo, fooInstance);
dc.register(IBar, async (deps) => new Bar(await deps.synthesize(...)));

// Synthesize a scope object with optional and required dependencies
const scope = dc.synthesize<IFoo, IProvideBar>(
  { IFoo },           // optional - may resolve to undefined
  { IBar }            // required - throws if not registered
);

// Access dependencies (all are Promises)
const foo = await scope.IFoo;    // IFoo | undefined
const bar = await scope.IBar;    // IBar (guaranteed)

// Check registration
dc.has(IFoo);                    // true
dc.has(IFoo, true);              // true, excluding parents
dc.unregister(IFoo);             // remove provider
```

### `IFluidDependencySynthesizer`

Interface for the dependency synthesizer, allowing abstraction over implementations.

### Type Helpers

- `FluidObjectSymbolProvider<T>` - Maps interface keys to themselves: `{ IFoo: "IFoo" }`
- `AsyncFluidObjectProvider<O, R>` - Synthesized scope type with Promise-wrapped properties
- `FluidObjectProvider<T>` - Union of all valid provider forms

## Architecture

### Hierarchical Resolution

Containers can have parent containers. Resolution checks the local container first, then traverses parents. Child registrations shadow parent registrations.

### Lazy Evaluation

All synthesized properties use lazy evaluation - the provider is not invoked until the property is accessed. This supports:

- Circular dependency resolution (via factory functions)
- Deferred initialization
- `LazyPromise` for explicit lazy loading

### Required vs Optional

- **Required**: Throws immediately during `synthesize()` if provider not found
- **Optional**: Returns a Promise that resolves to `undefined` if provider not found

## Testing

Tests use mocha and are in `src/test/dependencyContainer.spec.ts`. Key test patterns:

- All four provider types (value, promise, factory, async factory)
- Parent/child container resolution
- `LazyPromise` lazy evaluation verification
- Required provider missing throws
- Registration/unregistration lifecycle
