# @fluidframework/tree-agent-ses

SES (Secure ECMAScript) integration for `@fluidframework/tree-agent`. Provides a secure sandboxed environment for executing LLM-generated code that modifies SharedTree data.

## Purpose

When using `SharedTreeSemanticAgent`, the agent generates JavaScript code to modify tree data based on user queries. By default, this code runs via JavaScript's `Function` constructor, which has no security restrictions. This package provides `createSesEditExecutor()` which runs generated code in an SES compartment, preventing:

- Prototype pollution attacks
- Access to global objects and APIs
- Modifications to built-in objects

## Main Export

### `createSesEditExecutor<TSchema>(options?)`

Creates an `AsynchronousEditor<TSchema>` function for use with `SharedTreeSemanticAgent`.

```typescript
import { createSesEditExecutor } from "@fluidframework/tree-agent-ses/alpha";
import { SharedTreeSemanticAgent } from "@fluidframework/tree-agent/alpha";

const editor = createSesEditExecutor({
  lockdownOptions: {
    consoleTaming: "unsafe",  // Allow console access for debugging
    errorTaming: "unsafe",    // Preserve error stack traces
    stackFiltering: "verbose"
  },
  compartmentOptions: {
    globals: new Map([["myHelper", helperFn]])  // Additional globals
  }
});

const agent = new SharedTreeSemanticAgent(chatModel, treeView, { editor });
```

**Options:**
- `lockdownOptions` - Passed to SES `lockdown()`. Controls taming of built-ins.
- `compartmentOptions.globals` - `Map<string, unknown>` of additional globals available in the compartment. Note: `context` is reserved and cannot be overridden.

## Architecture

1. **First invocation**: Calls SES `lockdown()` to freeze all JavaScript intrinsics globally. This is idempotent - subsequent calls are no-ops.

2. **Each execution**: Creates a fresh SES `Compartment` with:
   - A `context` global (created via `createContext()` from tree-agent)
   - Any user-provided globals from `compartmentOptions`

3. **Code evaluation**: `compartment.evaluate(code)` runs the LLM-generated JavaScript in isolation.

## Key Implementation Details

- **Lockdown tracking**: Uses `Symbol.for("tree-agent.ses.locked")` on `globalThis` to track whether lockdown has been called, handling the case where SES was already locked down externally.

- **Context injection**: The `context` object provides the tree editing API (`context.root`, `context.create.*`, `context.is.*`, etc.) within the sandbox.

- **Error handling**: If SES blocks code (e.g., attempting to modify frozen objects), the error is caught and returned as an `EditResult` with `type: "editingError"`.

## Testing

Tests are in `src/test/ses.spec.ts` but are **skipped by default** (`describe.skip`) to avoid SES lockdown side effects in CI. SES lockdown is irreversible within a process, which can interfere with other tests.

To run SES tests locally:
1. Change `describe.skip` to `describe` in `ses.spec.ts`
2. Run `pnpm test:mocha` in isolation (not alongside other test suites)

Test coverage includes:
- Reserved `context` global protection
- Multiple executor creation without errors
- Custom globals passing through to compartment
- SES blocking forbidden operations (prototype pollution)
