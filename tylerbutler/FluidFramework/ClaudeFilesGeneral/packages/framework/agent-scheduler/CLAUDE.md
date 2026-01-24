# @fluidframework/agent-scheduler

Distributed task assignment system that ensures exactly one client runs each task across all connected container instances.

## Purpose

When multiple clients connect to the same Fluid container, some operations should only run once (not on every client). AgentScheduler uses consensus to assign tasks to specific clients, automatically reassigning when clients disconnect.

## Key Concepts

### Task Lifecycle

1. **Register** - Declare a task exists without volunteering to run it
2. **Pick** - Volunteer to run a task and provide a worker callback
3. **Assigned** - Consensus selects one client to run each task
4. **Release** - Voluntarily give up a task for others to claim
5. **Lost** - Task removed due to disconnect (auto-reassigned)

### Events

- `picked` - This client was assigned the task
- `released` - Task was successfully released back to the pool
- `lost` - Task lost due to disconnect or container attachment

## Main Exports

### `IAgentScheduler` (interface)

```typescript
interface IAgentScheduler {
  register(...taskUrls: string[]): Promise<void>;  // Declare tasks without volunteering
  pick(taskId: string, worker: () => Promise<void>): Promise<void>;  // Volunteer with callback
  release(...taskUrls: string[]): Promise<void>;  // Give up owned tasks
  pickedTasks(): string[];  // List tasks this client owns
}
```

### `AgentSchedulerFactory`

Factory for creating AgentScheduler instances as child data stores.

```typescript
// Register in your DataObjectFactory
registryEntries: new Map([AgentSchedulerFactory.registryEntry])

// Create instance
const scheduler = await AgentSchedulerFactory.createChildInstance(this.context);
```

### `TaskSubscription`

Helper class that simplifies monitoring a single task's ownership.

```typescript
const subscription = new TaskSubscription(agentScheduler, "myTask");
subscription.on("gotTask", () => { /* now the owner */ });
subscription.on("lostTask", () => { /* no longer owner */ });
subscription.volunteer();  // Opt in to running the task
subscription.haveTask();   // Check current ownership
```

## Architecture

- Uses `ConsensusRegisterCollection` for distributed agreement on task ownership
- Stores task-to-client mappings in a SharedMap with CRC handle
- Listens to quorum `removeMember` events to detect disconnects and clear orphaned tasks
- Non-interactive clients (summarizers) cannot pick tasks

### Internal State

- `registeredTasks` - Tasks this client has registered
- `locallyRunnableTasks` - Tasks this client can run (superset of running)
- `runningTasks` - Tasks currently assigned to and running on this client

### Connection Handling

- **Detached**: Uses temporary UUID-based clientId
- **Attached+Connected**: Uses runtime clientId, actively participates
- **Disconnected**: Clears running tasks, emits `lost` events, reacquires on reconnect

## Important Constraints

- Each task can only be registered once per client
- Each task can only be picked once per client
- Only currently running tasks can be released
- Only interactive clients can pick tasks
