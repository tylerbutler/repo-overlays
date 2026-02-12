---
name: effection-integration
description: Integrate effection structured concurrency into FluidFramework packages. Use when adding effection-based lifecycle management, replacing setTimeout/clearTimeout with scope-aware timers, bridging AbortSignal patterns with effection scopes, or introducing structured concurrency to a package that currently uses imperative dispose/close patterns. Triggers on mentions of effection, structured concurrency, EffectionScope, EffectionTimer, scope-aware cleanup, or scoped abort controllers in the FluidFramework repo.
---

# Effection Structured Concurrency Integration

Integrate effection (v4) into FluidFramework packages to replace ad-hoc lifecycle management with structured concurrency. The reference implementation is `packages/loader/container-loader/src/structuredConcurrency.ts`.

## Workflow

### 1. Identify candidates for structured concurrency

Look for these patterns in the target package:

- `setTimeout`/`clearTimeout` pairs — replace with `EffectionTimer`
- `new AbortController()` tied to a dispose method — replace with `createScopedAbortController`
- `new Promise(resolve => setTimeout(resolve, ms))` — replace with `createScopedDelay`
- Manual cleanup lists or `dispose()` methods that undo setup — replace with `EffectionScope.addCleanup`
- Event listener add/remove pairs across setup/teardown — register via `addCleanup`

### 2. Add the structuredConcurrency module

Copy `packages/loader/container-loader/src/structuredConcurrency.ts` into the target package's `src/` directory. Add `effection` as a dependency in the package's `package.json`.

See [references/patterns.md](references/patterns.md) for the full implementation with explanations.

### 3. Integrate into the class lifecycle

```ts
class MyService {
  private readonly _scope = new EffectionScope();

  setup(): void {
    // Register cleanup alongside setup
    element.addEventListener("event", this.handler);
    this._scope.addCleanup(() => element.removeEventListener("event", this.handler));
  }

  dispose(): void {
    // Single call tears down everything
    this._scope.close().catch(() => {});
  }
}
```

### 4. Replace timer patterns

Before:
```ts
private timer: ReturnType<typeof setTimeout> | undefined;
start() { this.timer = setTimeout(callback, ms); }
clear() { clearTimeout(this.timer); this.timer = undefined; }
// ...and don't forget to clearTimeout in dispose()!
```

After:
```ts
private readonly timer = new EffectionTimer(this._scope, ms, callback);
// start/clear/restart work the same, auto-cleaned on scope close
```

### 5. Replace AbortController patterns

Before:
```ts
private controller = new AbortController();
dispose() { this.controller.abort(); }
```

After:
```ts
private controller = createScopedAbortController(this._scope);
// auto-aborts on scope close
```

## References

- **[references/patterns.md](references/patterns.md)** — Full code for `EffectionScope`, `EffectionTimer`, `createScopedAbortController`, `createScopedDelay` with detailed explanations
- **[references/effection-api.md](references/effection-api.md)** — Quick reference for effection v4 APIs (`createScope`, `ensure`, `suspend`, `sleep`, `spawn`, `resource`, etc.)
