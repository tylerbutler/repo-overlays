# @fluid-experimental/attributor

## Purpose

This package provides operation attribution for Fluid Framework, enabling applications to track who made changes and when. Attribution maps operation sequence numbers to user identity and timestamp information, allowing features like "last edited by" displays or audit trails.

## Key Concepts

### Attribution Keys and Info

- **AttributionKey**: Identifies an operation (has a `type` of `"op"`, `"local"`, or `"detached"` and a `seq` number for sequenced ops)
- **AttributionInfo**: Contains `user` (IUser) and `timestamp` for each attributed operation

### Attributor Hierarchy

1. **IAttributor**: Base interface for mapping sequence numbers to AttributionInfo
2. **Attributor**: Simple in-memory implementation with a Map backing store
3. **OpStreamAttributor**: Extends Attributor to automatically record attribution from the op stream by listening to DeltaManager events

### Runtime Integration

- **IRuntimeAttributor**: Container-level API for querying attribution (supports `get`, `has`, `isEnabled`)
- **RuntimeAttributor**: Implementation that handles initialization from snapshots and summarization
- **RuntimeAttributorDataStoreChannel**: DataStore channel implementation for persistence

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              mixinAttributor()                       │
│   (Mixes attribution into ContainerRuntime)         │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│         RuntimeAttributorDataStoreChannel           │
│   (IFluidDataStoreChannel for persistence)          │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│              RuntimeAttributor                       │
│   (Manages OpStreamAttributor lifecycle)            │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│             OpStreamAttributor                       │
│   (Listens to ops, stores seq → AttributionInfo)    │
└─────────────────────────────────────────────────────┘
```

## Main Exports

```typescript
// Enable attribution in a container
import { mixinAttributor, getRuntimeAttributor } from "@fluid-experimental/attributor";

// Create a container runtime class with attribution support
const AttributingContainerRuntime = mixinAttributor(ContainerRuntime);

// Later, retrieve the attributor from a runtime
const attributor = await getRuntimeAttributor(runtime);
if (attributor?.isEnabled) {
  const info = attributor.get({ type: "op", seq: 123 });
  console.log(`Edited by ${info.user.id} at ${info.timestamp}`);
}
```

### Configuration

Attribution is enabled via feature flag in the container's monitoring context:

```typescript
// Set in loader options or feature gate config
"Fluid.Attribution.EnableOnNewFile": true
```

Only **new** documents will have attribution enabled. Existing documents without attribution data will not start tracking.

## Serialization

Attribution data is serialized efficiently using:

1. **StringInterner**: Deduplicates user objects by assigning integer IDs
2. **DeltaEncoder**: Stores timestamps as deltas from previous values (compact for sequential ops)
3. **LZ4 Compression**: Final compression pass using lz4js

The serialized format (`SerializedAttributor`):
```typescript
{
  interner: string[],      // Unique user JSON strings
  seqs: number[],          // Sequence numbers
  timestamps: number[],    // Delta-encoded timestamps
  attributionRefs: number[] // Indices into interner
}
```

## Testing

### Test Files

- `attributor.spec.ts`: Unit tests for `Attributor` class (basic get/entries operations)
- `opStreamAttributor.spec.ts`: Tests op stream listening and automatic attribution recording
- `attributorSerializer.spec.ts`: Round-trip serialization tests
- `deltaEncoder.spec.ts`: Timestamp delta encoding tests
- `stringInterner.spec.ts`: String interning tests
- `lz4Encoder.spec.ts`: LZ4 compression tests
- `attribution/sharedString.attribution.spec.ts`: Fuzz tests for SharedString attribution integration, includes snapshot comparison tests and size impact reports

### Running Tests

```bash
# From package directory
pnpm test:mocha

# Verbose output (shows size impact report)
pnpm test:mocha:verbose
```

### Test Data

The `src/test/attribution/documents/` directory contains generated test documents with various attribution storage strategies for snapshot comparison testing.

## Limitations

- **Local keys**: Attribution of local (unsequenced) operations is not yet supported
- **Detached keys**: Attribution of operations made before container attachment is not yet supported
- **Existing documents**: Attribution cannot be retroactively enabled on documents created without it
- **GC**: Garbage collection of stale attribution entries is marked as TODO
