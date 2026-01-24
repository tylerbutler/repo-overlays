# @fluidframework/request-handler

Simple request handling abstraction for routing requests within Fluid Framework container runtimes. Implements a chain-of-responsibility pattern where multiple handlers are tried in sequence until one can satisfy the request.

**Status: Deprecated** - This package is deprecated as part of the migration away from `IFluidRouter` to the "entryPoint" pattern. See `Removing-IFluidRouter.md` in the repo for migration guidance.

## Architecture

The package implements a straightforward handler chain pattern:

```
IRequest -> RequestParser -> Handler1 -> Handler2 -> ... -> HandlerN -> IResponse
                               |           |                   |
                               v           v                   v
                          undefined    undefined          404 Response
                          (continue)   (continue)         (if no handler matched)
```

### Key Components

**`RuntimeRequestHandler`** (type) - A function signature for request handlers:
- Receives a `RequestParser` (parsed request) and `IContainerRuntime`
- Returns `Promise<IResponse | undefined>`
- Returns `undefined` to pass to the next handler in the chain
- Returns an `IResponse` to stop the chain and return that response

**`buildRuntimeRequestHandler`** (function) - Factory that composes multiple handlers:
- Accepts variadic `RuntimeRequestHandler` arguments
- Returns a single request handler function `(IRequest, IContainerRuntime) => Promise<IResponse>`
- Handlers are called in order until one returns a response
- Returns a 404 response if no handler matches

## Main Exports

| Export | Type | Stability | Description |
|--------|------|-----------|-------------|
| `RuntimeRequestHandler` | type | `@legacy @beta` | Handler function signature |
| `buildRuntimeRequestHandler` | function | `@internal` | Composes handlers into a single handler |

### Export Paths

- `/` (public) - Empty (no public exports)
- `/legacy` - Exports `RuntimeRequestHandler` type
- `/internal` - Exports both `RuntimeRequestHandler` and `buildRuntimeRequestHandler`

## Usage Pattern

```typescript
import { buildRuntimeRequestHandler, RuntimeRequestHandler } from "@fluidframework/request-handler/internal";

const myHandler: RuntimeRequestHandler = async (request, runtime) => {
    if (request.pathParts[0] === "my-route") {
        // Handle the request
        return { status: 200, mimeType: "fluid/object", value: myObject };
    }
    // Pass to next handler
    return undefined;
};

const combinedHandler = buildRuntimeRequestHandler(
    myHandler,
    anotherHandler,
    // ... more handlers
);
```

## Dependencies

- `@fluidframework/container-runtime-definitions` - `IContainerRuntime` interface
- `@fluidframework/core-interfaces` - `IRequest`, `IResponse` interfaces
- `@fluidframework/runtime-utils` - `RequestParser`, `create404Response`

## Testing

Tests are located in `src/test/`. The current test file is a placeholder - the package has minimal logic and relies on the underlying `runtime-utils` for parsing and 404 response generation.
