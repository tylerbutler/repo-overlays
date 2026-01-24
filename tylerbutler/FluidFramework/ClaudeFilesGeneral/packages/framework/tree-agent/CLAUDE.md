# @fluidframework/tree-agent

Experimental package for integrating AI/LLM agents with SharedTree. Enables AI models to analyze and edit tree data structures through natural language queries.

## Purpose

This package provides infrastructure for creating AI agents that can:
- Understand SharedTree schema and current state
- Generate JavaScript code to edit tree data based on user prompts
- Handle edit failures gracefully with automatic rollback
- Track tree changes between queries

## Architecture

### Core Components

**SharedTreeSemanticAgent** - Main agent class that:
- Wraps an LLM client (`SharedTreeChatModel`) and a tree view
- Generates system prompts explaining tree schema and editing APIs
- Manages branching for safe edits (fork before edit, merge on success, discard on failure)
- Tracks external tree changes and notifies the model

**SharedTreeChatModel** - Interface for LLM integration:
- `query(message)` - Process user query, optionally calling `edit()` to modify tree
- `appendContext(text)` - Receive schema/state information
- `editToolName` - Name of the tool the LLM should use for edits

**Context** - Object passed to generated edit code:
- `root` - Read/write access to tree root
- `create.*` - Builder functions for each schema type
- `is.*` - Type guards for each schema type
- `isArray()`, `isMap()` - Type checks for collections
- `parent()`, `key()` - Navigation helpers

### Branching Strategy

The agent uses SharedTree's branching for transactional edits:
1. Fork the tree before each query
2. Apply edits to the fork
3. On success: merge fork back to main branch
4. On failure: discard fork, tree unchanged

### Type Systems for Schema Binding

Two approaches for exposing methods/properties to the LLM:

**Zod-based** - Strict compile-time type checking:
```typescript
methods.expose(MyClass, "methodName", buildFunc(
  { returns: z.boolean() },
  ["arg1", z.string()],
  ["arg2", z.number()]
));
```

**TypeFactory-based** - Runtime type descriptions:
```typescript
methods.expose(MyClass, "methodName", buildFunc(
  { returns: typeFactory.boolean() },
  ["arg1", typeFactory.string()]
));
```

## Key Exports

| Export | Description |
|--------|-------------|
| `SharedTreeSemanticAgent` | Main agent class |
| `createContext` | Create editing context for custom editors |
| `SharedTreeChatModel` | Interface for LLM clients |
| `exposeMethodsSymbol` | Symbol for exposing class methods |
| `exposePropertiesSymbol` | Symbol for exposing class properties |
| `buildFunc` | Helper for defining method signatures |
| `typeFactory` | Runtime type factory for schema binding |
| `llmDefault` | Symbol for field defaults populated by functions |

## Usage Pattern

```typescript
// 1. Define schema
class Task extends sf.object("Task", { title: sf.string, done: sf.boolean }) {}

// 2. Create tree view
const view = independentView(new TreeViewConfiguration({ schema: Task }));
view.initialize({ title: "Example", done: false });

// 3. Create LLM client (implements SharedTreeChatModel)
const model: SharedTreeChatModel = {
  editToolName: "EditTree",
  appendContext(text) { /* add to LLM context */ },
  async query({ text, edit }) {
    // Call your LLM, parse response, call edit() with generated JS
    const result = await edit(`context.root.done = true;`);
    return result.message;
  }
};

// 4. Create agent and query
const agent = new SharedTreeSemanticAgent(model, view);
const response = await agent.query("Mark the task as complete");
```

## Testing

Tests use mock `SharedTreeChatModel` implementations to verify:
- Edit application and rollback behavior
- Error handling for malformed/failing code
- Sequential edit limits
- Context helper functions (`create`, `is`, `parent`, `key`)
- Subtree editing (agent on nested node rather than root)

Run tests with `pnpm test:mocha` from this package directory.

## Implementation Notes

- **Edit execution**: Default editor uses dynamic code evaluation. Custom `editor` option allows sandboxed execution (e.g., SES via `@fluidframework/tree-agent-ses`).
- **Schema rendering**: Tree schema is converted to TypeScript interface definitions in the system prompt.
- **Array/Map APIs**: Special mutation APIs explained to LLM since standard JS methods don't work on tree collections.
- **Node reinsertion**: Removed nodes cannot be reinserted; must be deep-cloned via `context.create.*`.
- **Sequential edit limit**: Default 20 edits per query to prevent infinite loops.
