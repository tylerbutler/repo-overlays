# Effection API Quick Reference

Package: `effection` (v4.x)

## Core Concepts

Effection uses **generator functions** as Operations. `yield*` composes operations (like `await` for promises). All child tasks are automatically cleaned up when their parent scope exits — this is structured concurrency.

## Key APIs

### `createScope(): [Scope, destroy]`
Creates a standalone scope for use outside operations. Returns a tuple of the scope and a destroy function. This is what `EffectionScope` wraps.

### `Scope.run(operation): Task<T>`
Run an operation within a scope from outside effection. Returns a `Task<T>` (which is also a `Promise<T>` and an `Operation<T>`).

### `spawn(operation): Operation<Task<T>>`
Create a child task within an operation. Child tasks are halted when their parent completes.

### `sleep(ms): Operation<void>`
Cooperative sleep — automatically cancelled when scope closes.

### `ensure(fn): Operation<void>`
Register cleanup that runs when the current operation shuts down (like `finally`). Accepts `() => Operation<unknown> | void` — use a plain function for synchronous cleanup.

### `suspend(): Operation<never>`
Suspend the current operation indefinitely. Used to keep a task alive until its scope closes.

### `useScope(): Operation<Scope>`
Capture the current scope from within an operation. Useful for bridging back into effection from callbacks.

### `useAbortSignal(): Operation<AbortSignal>`
Create an AbortSignal bound to the current scope. Automatically aborts when scope exits.

### `call(fn): Operation<T>`
Wrap an async function or Promise into an Operation with proper scope management.

### `resource(fn): Operation<T>`
Define a managed resource with automatic cleanup (provide/finally pattern).

### `Task<T>`
Extends `Future<T>` extends `Operation<T>` & `Promise<T>`. Has `.halt(): Future<void>` to cancel.

## Common Patterns

**Fire-and-forget within a scope:**
```ts
scope.run(function* () { /* task owned by scope */ });
```

**Register cleanup on scope exit:**
```ts
scope.run(function* () {
  yield* ensure(() => { cleanup(); });
  yield* suspend();
});
```

**Scope-aware timer:**
```ts
scope.run(function* () {
  yield* sleep(ms);
  callback();
});
```
