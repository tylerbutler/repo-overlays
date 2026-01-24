# @fluidframework/fluid-telemetry

Customer-facing telemetry system for Fluid Framework applications. Transforms raw container system events into structured, consumable telemetry and routes them to configurable consumers.

## Purpose

This package provides a telemetry abstraction layer that:
- Listens to `IFluidContainer` system events and converts them into well-typed telemetry objects
- Routes telemetry to pluggable consumers (e.g., Azure Application Insights)
- Tracks container instances with unique identifiers

## Architecture

The package follows a **Producer-Manager-Consumer** pattern:

```
IFluidContainer Events → TelemetryManager → TelemetryProducer → ITelemetryConsumer[]
     (connected,           (orchestrates)     (transforms to      (sends to destination)
      disconnected,                            IContainerTelemetry)
      disposed)
```

### Key Components

**`ContainerTelemetryManager`** (`container/telemetryManager.ts`)
- Subscribes to raw `IFluidContainer` events (`connected`, `disconnected`, `disposed`)
- Routes events to the producer, then distributes resulting telemetry to all consumers
- Emits heartbeat telemetry every 60 seconds when container is connected

**`ContainerEventTelemetryProducer`** (`container/telemetryProducer.ts`)
- Transforms raw system events into typed `IContainerTelemetry` objects
- Generates unique `containerInstanceId` (UUID) per container load to distinguish instances

**`ITelemetryConsumer`** (`common/consumers/index.ts`)
- Interface for consuming telemetry events
- Implement `consume(event: IFluidTelemetry)` to handle events

**`AppInsightsTelemetryConsumer`** (`app-insights/appInsightsTelemetryConsumer.ts`)
- Built-in consumer that sends telemetry to Azure Application Insights

## Main Exports

### Entry Point Function

```typescript
import { startTelemetry, TelemetryConfig } from "@fluidframework/fluid-telemetry/beta";

startTelemetry({
  container: fluidContainer,
  containerId: "your-container-id",
  consumers: [new AppInsightsTelemetryConsumer(appInsightsClient)]
});
```

### Telemetry Types

| Type | Event Name | Description |
|------|------------|-------------|
| `ContainerConnectedTelemetry` | `fluidframework.container.connected` | Container connected to service |
| `ContainerDisconnectedTelemetry` | `fluidframework.container.disconnected` | Container disconnected |
| `ContainerDisposedTelemetry` | `fluidframework.container.disposed` | Container closed (includes optional error) |

### Telemetry Structure

All container telemetry extends `IContainerTelemetry`:
- `eventName`: Scoped event name (e.g., `fluidframework.container.connected`)
- `containerId`: Stable ID across clients/loads
- `containerInstanceId`: Unique per container load (UUID)

## API Tiers

- **`/beta`**: Main consumer API (`startTelemetry`, `TelemetryConfig`, all telemetry types)
- **`/internal`**: Internal implementation details

## Testing

### Unit Tests (`containerTelemetry.spec.ts`)
- Uses `MockFluidContainer` (TypedEventEmitter) to simulate container events
- Spies on `AppInsightsTelemetryConsumer` to verify telemetry structure
- Tests each event type: connected, disconnected, disposed (with/without error)

### End-to-End Tests (`containerTelemetryEndToEnd.spec.realsvc.ts`)
- Requires Tinylicious server running on port 7070
- Creates real `IFluidContainer` instances via `TinyliciousClient`
- Verifies actual container lifecycle events produce correct telemetry

```bash
# Run unit tests only
pnpm test:mocha:esm:unit

# Run E2E tests (requires Tinylicious)
pnpm test:realsvc:tinylicious
```

## Implementation Notes

- **Heartbeat**: Synthetic `fluidframework.container.heartbeat` events are emitted every 60 seconds while connected (not publicly typed)
- **Type safety**: Uses `satisfies` for telemetry objects to ensure type conformance
- **Error propagation**: `ContainerDisposedTelemetry.error` includes `ICriticalContainerError` when container closes due to error
