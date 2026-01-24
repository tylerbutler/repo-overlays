# CLAUDE.md - @fluid-experimental/oldest-client-observer

## Purpose

Utility to determine if the local client is the "oldest" (first-connected) among all connected clients in a Fluid session. This is useful for leader election scenarios where exactly one client should perform a task (e.g., background processing, cleanup operations).

**Status**: Experimental (`@fluid-experimental`). APIs may change without notice.

## Key Concepts

### "Oldest" Client Definition

The oldest client is determined by comparing `sequenceNumber` values from the quorum. The client with the lowest sequence number (i.e., connected earliest) is considered oldest.

### Special Cases

- **Detached container**: Local client is always oldest (only client that knows about it)
- **Disconnected**: Cannot be oldest (not participating in quorum)
- **Readonly mode**: Cannot be oldest (clientId not in quorum)

## Architecture

```
IOldestClientObservable (input)     OldestClientObserver (output)
├── getQuorum()                     ├── isOldest(): boolean
├── attachState                     └── Events:
├── connected                           ├── "becameOldest"
└── clientId                            └── "lostOldest"
```

The `IOldestClientObservable` interface is satisfied by both `IContainerRuntime` and `IFluidDataStoreRuntime`.

## Main Exports

All exports available via `/legacy` entry point (beta stability).

| Export | Type | Description |
|--------|------|-------------|
| `OldestClientObserver` | Class | Main implementation |
| `IOldestClientObserver` | Interface | Observer contract with `isOldest()` method |
| `IOldestClientObservable` | Interface | Input contract (runtime compatibility) |
| `IOldestClientObserverEvents` | Interface | Events: `becameOldest`, `lostOldest` |
| `IOldestClientObservableEvents` | Interface | Events: `connected`, `disconnected` |

## Usage

```typescript
import { OldestClientObserver } from "@fluid-experimental/oldest-client-observer/legacy";

// From container runtime
const observer = new OldestClientObserver(containerRuntime);

// From data store runtime
const observer = new OldestClientObserver(this.runtime);

// Check status
if (observer.isOldest()) {
    // Perform leader-only task
}

// React to changes
observer.on("becameOldest", () => { /* start leader tasks */ });
observer.on("lostOldest", () => { /* stop leader tasks */ });
```

## Implementation Details

- Listens to quorum `addMember`/`removeMember` events
- Listens to runtime `connected`/`disconnected` events
- Recomputes oldest status on any change and emits events only when status changes
- Uses `TypedEventEmitter` from `@fluid-internal/client-utils`

## Testing

This package has no tests. The implementation is straightforward (< 100 lines) and relies on well-tested quorum mechanics.

## Caveats

When using with `IFluidDataStoreRuntime`, the "oldest" determination is scoped to knowledge about that specific data store, which may differ from the container-level oldest client for detached data stores.
