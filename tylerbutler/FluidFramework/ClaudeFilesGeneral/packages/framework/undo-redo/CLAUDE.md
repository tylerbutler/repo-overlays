# @fluidframework/undo-redo

In-memory undo/redo stack infrastructure for Fluid Framework distributed data structures.

## Purpose

Provides an `UndoRedoStackManager` that tracks reversible operations as a stack-of-stacks, plus handlers that automatically create `IRevertible` objects for SharedMap and SharedSegmentSequence changes.

## Architecture

### Core Components

**UndoRedoStackManager** - Central coordinator managing undo/redo stacks
- Stack-of-stacks structure: outer stack = operations, inner stack = `IRevertible` objects per operation
- `pushToCurrentOperation(revertible)` - Add a revertible to current operation
- `closeCurrentOperation()` - Mark current operation complete (controls undo granularity)
- `undoOperation()` / `redoOperation()` - Execute undo/redo
- `on("changePushed", callback)` - Event when changes are pushed to either stack

**IRevertible** - Interface for reversible changes
```typescript
interface IRevertible {
  revert(): void;   // Reverse the change
  discard(): void;  // Clean up when removed from redo stack
}
```

### Built-in Handlers

**SharedMapUndoRedoHandler** - For SharedMap
- Attach via `handler.attachMap(map)` / `detachMap(map)`
- Creates `SharedMapRevertible` for each local `valueChanged` event
- Simple implementation: stores previous value, reverts by calling `map.set(key, previousValue)`

**SharedSegmentSequenceUndoRedoHandler** - For SharedString and other sequences
- Attach via `handler.attachSequence(sequence)` / `detachSequence(sequence)`
- Creates `SharedSegmentSequenceRevertible` for local `sequenceDelta` events
- Uses `TrackingGroup` from merge-tree to track segments across splits/merges
- Batches changes within an operation for efficiency

### SharedMatrix Integration

SharedMatrix has its own built-in undo support that integrates with `UndoRedoStackManager`:
```typescript
const undo = new UndoRedoStackManager();
matrix.openUndo(undo);  // Matrix pushes IRevertibles directly
```

## Usage Pattern

```typescript
// 1. Create stack manager
const undoRedo = new UndoRedoStackManager();

// 2. Attach handlers to DDSs
const mapHandler = new SharedMapUndoRedoHandler(undoRedo);
mapHandler.attachMap(mySharedMap);

const seqHandler = new SharedSegmentSequenceUndoRedoHandler(undoRedo);
seqHandler.attachSequence(mySharedString);

// 3. Control operation granularity (e.g., on word boundaries)
undoRedo.closeCurrentOperation();

// 4. Undo/redo
undoRedo.undoOperation();  // Returns false if nothing to undo
undoRedo.redoOperation();  // Returns false if nothing to redo
```

## Key Behaviors

- **Redo stack clears** when new changes are pushed (standard undo/redo semantics)
- **Operation boundaries** are controlled by the consumer via `closeCurrentOperation()`
- **Local changes only** - handlers ignore remote changes (undo is per-client)
- **Entirely in-memory** - does not persist or affect other clients

## Memory Considerations

For sequences, `TrackingGroup` usage has overhead:
- Removed segments in a TrackingGroup won't be garbage collected
- Segments can only merge if they share the same TrackingGroups

This is minimized by batching changes into fewer `IRevertible` objects.

## Testing

Tests use `MockFluidDataStoreRuntime` and `MockContainerRuntimeFactory` from `@fluidframework/test-runtime-utils`. Key test scenarios:
- Insert/delete sequences with chunked operations
- Undo/redo across segment splits
- Matrix cell operations
