# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`@fluidframework/presence` is a Fluid Framework package for lightweight session-focused data sharing. It enables real-time presence features (like cursors, selections, user states) that don't require persistent storage—data exists only while clients are connected.

## Architecture

### Entry Point Pattern

The package uses tiered exports via `package.json` exports field:
- `/beta` - Stable beta APIs (primary import path)
- `/alpha` - Alpha APIs under development
- `/legacy/alpha` - Legacy alpha APIs

### Core Components

**PresenceManager** (`presenceManager.ts`) - Central orchestrator that:
- Manages attendee lifecycle (join/disconnect)
- Routes signals between clients via `PresenceDatastoreManager`
- Coordinates workspaces through the `states` and `notifications` properties

**StateFactory** (`stateFactory.ts`) - Factory for creating state objects:
- `StateFactory.latest()` - Creates `Latest<T>` for single values per attendee
- `StateFactory.latestMap()` - Creates `LatestMap<T,K>` for key-value maps per attendee

**Value Managers** - State object implementations:
- `LatestValueManagerImpl` (`latestValueManager.ts`) - Tracks latest value per attendee
- `LatestMapValueManagerImpl` (`latestMapValueManager.ts`) - Tracks key-value maps per attendee

**PresenceStatesImpl** (`presenceStates.ts`) - Manages workspaces:
- Groups related state objects under a workspace address
- Handles schema registration and state updates

**SystemWorkspace** (`systemWorkspace.ts`) - Internal workspace tracking:
- Manages `SessionClient` instances (attendees)
- Maps client connection IDs to stable attendee IDs

**Protocol** (`protocol.ts`) - Signal message types:
- `joinMessageType` - Client join announcements
- `datastoreUpdateMessageType` - State updates
- `acknowledgementMessageType` - Join acknowledgements

### Data Flow

```
Local State Change → StateDatastore.localUpdate() →
PresenceDatastoreManager.enqueueMessage() → (throttled) →
IEphemeralRuntime.submitSignal() → Other Clients →
processSignal() → PresenceStatesImpl.processUpdate() →
Value Manager.update() → Events fired
```

### Throttling

Updates are batched via `allowableUpdateLatencyMs` (default 60ms). Notifications bypass throttling but may be grouped with pending updates.

## Testing

Tests use `MockEphemeralRuntime` (`src/test/mockEphemeralRuntime.ts`) which:
- Simulates connected clients via `MockAudience` and `MockQuorumClients`
- Tracks expected signals via `signalsExpected` array
- Provides `connect()`, `disconnect()`, `removeMember()` for simulating client lifecycle

Test pattern:
```typescript
const runtime = new MockEphemeralRuntime();
const presence = createPresenceManager(runtime);
const workspace = presence.states.getWorkspace("name:test", {
    myState: StateFactory.latest({ local: initialValue })
});
```

## Key Types

- `Presence` - Main interface for accessing states and notifications workspaces
- `Attendee` - Represents a connected client with stable `attendeeId`
- `Latest<T>` / `LatestRaw<T>` - Single-value state with optional validator
- `LatestMap<T,K>` / `LatestMapRaw<T,K>` - Map-based state with optional validators
- `StatesWorkspace` - Groups related state objects
- `NotificationsWorkspace` - Groups notification managers
