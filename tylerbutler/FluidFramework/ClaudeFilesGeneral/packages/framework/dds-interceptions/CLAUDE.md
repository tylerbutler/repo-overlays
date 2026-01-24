# @fluid-experimental/dds-interceptions

Wrapper utilities that intercept DDS (Distributed Data Structure) mutation operations to execute custom callbacks atomically alongside the original operation.

## Purpose

This package enables automatic side effects when DDS values are modified. The primary use case is **user attribution** - automatically recording who made each change. When a value is set on the wrapped DDS, both the original operation and the callback execute within `orderSequentially()`, ensuring they are batched as a single atomic transaction.

## Architecture

The package uses **prototype-based wrapping**: each `create*WithInterception` function creates a new object that inherits from the underlying DDS via `Object.create()`, then overrides specific mutation methods. This allows:

- Read operations to pass through unchanged to the underlying DDS
- Write operations to be intercepted and wrapped with callbacks
- Full bidirectional visibility between wrapper and underlying DDS

### Recursion Protection

Each wrapper tracks an `executingCallback` flag to detect and assert if the callback attempts to call the wrapped object's mutation methods, which would cause infinite recursion.

## Exports

### `createSharedMapWithInterception`

```typescript
function createSharedMapWithInterception(
  sharedMap: ISharedMap,
  context: IFluidDataStoreContext,
  setInterceptionCallback: (sharedMap: ISharedMap, key: string, value: unknown) => void,
): ISharedMap
```

Intercepts the `set()` method. The callback receives the underlying map, key, and value.

**Example - User Attribution:**
```typescript
const wrapped = createSharedMapWithInterception(map, context, (map, key, value) => {
  map.set(`${key}.attribution`, { userId: currentUser.id });
});
wrapped.set("color", "blue"); // Also sets "color.attribution"
```

### `createDirectoryWithInterception`

```typescript
function createDirectoryWithInterception<T extends IDirectory>(
  baseDirectory: T,
  context: IFluidDataStoreContext,
  setInterceptionCallback: (
    baseDirectory: IDirectory,
    subDirectory: IDirectory,
    key: string,
    value: unknown,
  ) => void,
): T
```

Intercepts `set()` and also wraps subdirectory operations (`createSubDirectory`, `getSubDirectory`, `getWorkingDirectory`, `subdirectories`) so all nested directories are similarly intercepted.

The callback receives both the **base directory** (root of the wrapped hierarchy) and the **subdirectory** where the set occurred, enabling different attribution strategies:
- Mirror structure: Store attribution in a parallel `/attribution/...` tree
- Inline structure: Store attribution in each directory's `/attribution` subdirectory

### `createSharedStringWithInterception`

```typescript
function createSharedStringWithInterception(
  sharedString: SharedString,
  context: IFluidDataStoreContext,
  propertyInterceptionCallback: (props?: PropertySet) => PropertySet,
): SharedString
```

Intercepts property-modifying methods and transforms properties before they reach the underlying SharedString. Unlike the Map/Directory wrappers, this uses a **property transformation** pattern rather than a side-effect callback.

**Intercepted methods:**
- `insertText`, `insertTextRelative`
- `insertMarker`, `insertMarkerRelative`
- `replaceText`
- `annotateRange`, `annotateMarker`
- `insertAtReferencePosition`

**Example - Adding User Attribution to Properties:**
```typescript
const wrapped = createSharedStringWithInterception(str, context, (props) => ({
  ...props,
  userId: currentUser.id,
}));
wrapped.insertText(0, "Hello", { style: "bold" });
// Properties become: { style: "bold", userId: "..." }
```

## Testing

Tests use `MockFluidDataStoreRuntime` with a mock `orderSequentially` that simply executes the callback immediately. Tests verify:

1. **Interception works**: Callbacks execute and can modify the DDS
2. **Bidirectional visibility**: Changes via wrapper are visible on underlying DDS and vice versa
3. **Recursion detection**: Calling wrapped methods from callback asserts but leaves object usable
4. **Subdirectory wrapping** (Directory only): Created/retrieved subdirectories are also wrapped

Run tests with:
```bash
pnpm test:mocha
```

## Implementation Notes

- All wrappers require `IFluidDataStoreContext` to access `containerRuntime.orderSequentially()`
- The `orderSequentially` batching ensures callback operations are part of the same op batch
- Callbacks should operate on the **underlying** DDS (passed to callback), not the wrapper, to avoid recursion
- Changes made directly on the underlying DDS bypass interception entirely
