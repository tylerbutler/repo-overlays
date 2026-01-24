# @fluidframework/react

React integration utilities for SharedTree, providing hooks and components that automatically re-render when tree content changes.

## Purpose

This package solves a core challenge: ensuring React components re-render when SharedTree content changes. Reading tree node properties directly in React props creates "invalidation bugs" where edits don't trigger re-renders. This package provides type-safe patterns to prevent these bugs.

## Key Concepts

### The Invalidation Problem

```tsx
// BUG: Reading `item.text` from props won't re-render when text changes
const ItemBad = ({ item }: { item: Item }) => <span>{item.text}</span>;

// CORRECT: Wrap with observation tracking
const ItemGood = withTreeObservations(
  ({ item }: { item: Item }) => <span>{item.text}</span>
);
```

### PropTreeNode - Type-Safe Node References

`PropTreeNode<T>` is a type-erased wrapper that prevents direct property access at compile time, forcing reads through observation-tracked hooks.

```tsx
// PropTreeNode removes direct property access - this is a compile error:
const BadParent = ({ item }: { item: PropTreeNode<Item> }) => (
  <span>{item.text}</span>  // Error: Property 'text' does not exist
);

// Must use hooks to read content:
const GoodParent = ({ item }: { item: PropTreeNode<Item> }) => (
  <ChildWithObservations item={item} />
);
```

## Main Exports

### Hooks

| Hook | Purpose |
|------|---------|
| `useTree(node)` | Invalidates on any change in subtree (simple but coarse) |
| `useTreeObservations(fn)` | Tracks actual reads, only invalidates on observed changes |
| `usePropTreeNode(propNode, fn)` | Unwraps PropTreeNode and tracks observations |
| `usePropTreeRecord(record, fn)` | Same as above for records of nodes |

### Higher-Order Components

| HOC | Purpose |
|-----|---------|
| `withTreeObservations(Component)` | Wraps component to track tree observations |
| `withMemoizedTreeObservations(Component)` | Same + React.memo for array child reuse |

### DataObject Integration

| Export | Purpose |
|--------|---------|
| `treeDataObject(config, initializer)` | Creates schema-aware DataObject for ContainerSchema |
| `TreeViewComponent` | Handles schema compatibility, displays errors or content |

### Type Utilities

| Type | Purpose |
|------|---------|
| `PropTreeNode<T>` | Type-erased TreeNode for safe prop passing |
| `WrapNodes<T>` | Recursively wraps TreeNodes in PropTreeNode |
| `PropTreeNodeRecord` | Record type with PropTreeNode values |

### Utility Functions

| Function | Purpose |
|----------|---------|
| `toPropTreeNode(node)` | Convert TreeNode to PropTreeNode |
| `unwrapPropTreeNode(propNode)` | Extract TreeNode (use in event handlers) |
| `objectIdNumber(obj)` | Stable numeric ID for React list keys |

## Usage Patterns

### Basic Component with Observations

```tsx
const ItemView = withTreeObservations(({ item }: { item: Item }) => (
  <div>
    <span>{item.name}</span>
    <button onClick={() => item.count++}>Increment</button>
  </div>
));
```

### Parent-Child with PropTreeNode

```tsx
// Parent passes PropTreeNode - cannot accidentally read properties
const List = withTreeObservations(({ items }: { items: Items }) => (
  <ul>
    {items.map(item => (
      <ItemView key={objectIdNumber(item)} item={toPropTreeNode(item)} />
    ))}
  </ul>
));

// Child unwraps and reads with tracking
const ItemView = ({ item }: { item: PropTreeNode<Item> }) => {
  const { name } = usePropTreeNode(item, node => ({ name: node.name }));
  return <li>{name}</li>;
};
```

### Array with Memoized Children

```tsx
const ItemView = withMemoizedTreeObservations(({ item }: { item: Item }) => (
  <span>{item.x}</span>
));

const ListView = withTreeObservations(({ list }: { list: List }) => (
  <div>
    {list.map(item => (
      <ItemView key={objectIdNumber(item)} item={item} />
    ))}
  </div>
));
// When items are added/removed, only new items render; existing items reuse
```

### DataObject in Container Schema

```tsx
const schema = {
  initialObjects: {
    inventory: treeDataObject(
      new TreeViewConfiguration({ schema: Inventory }),
      () => new Inventory({ nuts: 5, bolts: 6 })
    ),
  },
} satisfies ContainerSchema;
```

## Testing

Tests use `@testing-library/react` with `global-jsdom` for DOM simulation. Key test patterns:

- **StrictMode testing**: All tests run both with and without React StrictMode to catch double-render issues
- **Render logging**: Tests track render calls to verify correct invalidation behavior
- **Unmount cleanup**: Tests verify subscriptions are cleaned up via FinalizationRegistry (requires `--expose-gc`)

```bash
# Run tests (requires Node with GC exposure for cleanup tests)
pnpm test:mocha
```

## Implementation Notes

### Observation Tracking

Uses `TreeAlpha.trackObservationsOnce` internally to track which tree properties are read during render, then subscribes to change events only for those specific observations.

### Cleanup Strategy

Event subscriptions created during render are cleaned up via:
1. **Re-render**: Previous subscriptions cleared before new tracking
2. **Unmount**: `FinalizationRegistry` calls unsubscribe when React releases state object

This avoids React's useEffect for subscription management, which would have timing gaps between render and effect execution where changes could be missed.

### WrapNodes Type

`WrapNodes<T>` recursively wraps TreeNodes but carefully avoids breaking:
- Nominal types (classes with private/protected members)
- Constructors and functions
- Map/Set types (leaves them unwrapped)
