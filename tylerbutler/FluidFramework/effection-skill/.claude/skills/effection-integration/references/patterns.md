# Effection Structured Concurrency Patterns

Reference implementation: `packages/loader/container-loader/src/structuredConcurrency.ts`

## EffectionScope — Imperative wrapper over effection's scope

Bridges effection's cooperative structured concurrency with class-based lifecycle patterns (construct/dispose). A scope owns tasks and cleanup functions; closing the scope tears everything down.

```ts
import { createScope, ensure, suspend, sleep, type Operation, type Scope, type Task } from "effection";

type CleanupFn = () => void;

export class EffectionScope {
  private readonly scope: Scope;
  private readonly destroy: () => Promise<void>;
  private closed = false;

  public constructor() {
    [this.scope, this.destroy] = createScope();
  }

  public run<T>(operation: () => Operation<T>): Task<T> {
    return this.scope.run(operation);
  }

  public addCleanup(cleanup: CleanupFn): void {
    // Task is intentionally spawned into the scope's lifetime; awaiting it would
    // block forever (it calls suspend()). The scope owns the task and tears it down on close().
    // eslint-disable-next-line @typescript-eslint/no-floating-promises
    this.scope.run(function* () {
      yield* ensure(() => { cleanup(); });
      yield* suspend();
    });
  }

  public async close(): Promise<void> {
    if (this.closed) return;
    this.closed = true;
    await this.destroy();
  }
}
```

Key points:
- `createScope()` returns `[scope, destroy]`. The destroy function tears down the scope.
- `addCleanup` spawns a task that registers an `ensure` handler then suspends forever. When the scope closes, the task is halted and `ensure` runs the cleanup.
- `ensure()` accepts `() => Operation<unknown> | void`. For synchronous cleanup, pass a plain function — NOT a generator (avoids `require-yield` lint error).
- `scope.run()` returns a `Task<T>` which extends `Promise<T>`. In `addCleanup`, the returned promise is intentionally not awaited (fire-and-forget within the scope's lifetime).

## EffectionTimer — Scope-aware setTimeout replacement

Replaces `setTimeout`/`clearTimeout` patterns. Timers auto-cancel when scope closes.

```ts
export class EffectionTimer {
  private task: Task<void> | undefined;

  public constructor(
    private readonly scope: EffectionScope,
    private readonly defaultTimeoutMs: number,
    private readonly defaultCallback: () => void,
  ) {}

  public get hasTimer(): boolean {
    return this.task !== undefined;
  }

  public start(
    timeoutMs: number = this.defaultTimeoutMs,
    callback: () => void = this.defaultCallback,
  ): void {
    this.clear();
    this.task = this.scope.run(function* () {
      yield* sleep(timeoutMs);
      callback();
    });
  }

  public restart(
    timeoutMs: number = this.defaultTimeoutMs,
    callback: () => void = this.defaultCallback,
  ): void {
    this.start(timeoutMs, callback);
  }

  public clear(): void {
    if (this.task === undefined) return;
    // We only need to initiate the halt; the scope's structured concurrency
    // guarantees proper teardown regardless of whether we await the result.
    // eslint-disable-next-line @typescript-eslint/no-floating-promises
    this.task.halt();
    this.task = undefined;
  }
}
```

Key points:
- `sleep()` is effection's cooperative sleep — automatically cancelled when scope closes.
- `task.halt()` returns `Future<void>` (extends `Promise<void>`). In a synchronous `clear()` method, we initiate the halt without awaiting.
- Starting a timer clears any existing one first.

## Bridge Utilities

### createScopedAbortController — AbortSignal bridge

Bridges effection's cooperative cancellation with `AbortSignal`-based APIs.

```ts
export function createScopedAbortController(scope: EffectionScope): AbortController {
  const controller = new AbortController();
  scope.addCleanup(() => {
    if (!controller.signal.aborted) {
      controller.abort("Scope closed");
    }
  });
  return controller;
}
```

### createScopedDelay — Scope-aware delay promise

Replaces `new Promise(resolve => setTimeout(resolve, delayMs))` with a version that rejects on scope closure.

```ts
export async function createScopedDelay(
  scope: EffectionScope,
  delayMs: number,
): Promise<void> {
  return new Promise<void>((resolve, reject) => {
    const timeoutId = setTimeout(resolve, delayMs);
    scope.addCleanup(() => {
      clearTimeout(timeoutId);
      reject(new Error("Delay cancelled by scope closure"));
    });
  });
}
```

## Integration pattern in a class with lifecycle

```ts
class MyContainer {
  private readonly _effectionScope = new EffectionScope();

  // Use scope for cleanup registration
  private setup(): void {
    document.addEventListener("visibilitychange", this.handler);
    this._effectionScope.addCleanup(() => {
      document.removeEventListener("visibilitychange", this.handler);
    });
  }

  // Close scope during dispose/close
  private dispose(): void {
    this._effectionScope.close().catch(() => {});
    this.removeAllListeners();
  }
}
```

Key points:
- Create scope as a class field (constructed once, lives for the object's lifetime).
- Register safety-net cleanups via `addCleanup` for things like DOM listeners.
- Close the scope in both `close()` and `dispose()` paths.
- `.catch(() => {})` on `close()` since it's called from a synchronous dispose context.
