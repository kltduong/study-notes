# Callbacks

**TL;DR:** A callback is a function passed to another function to be called later. Callbacks can be synchronous (called immediately, like `Array.map`) or asynchronous (deferred to the runtime, like `setTimeout`). The receiving function's implementation determines which. Error handling in async callbacks requires the error-first convention since `try/catch` can't cross async boundaries.

## What Is a Callback?

A function you pass as an argument so someone else can invoke it. The term describes the **role**, not anything special about the function itself. Any function can be a callback.

```js
function doSomething(callback) {
  callback("done");
}
doSomething((result) => console.log(result));
```

The key: **you don't call it — the receiver does**, at whatever time and with whatever arguments it decides.

## Sync vs Async Callbacks

"Callback" does not mean "async." It's a calling convention that can go either way.

**Synchronous** — called during the outer function's execution, on the same call stack:

```js
[1, 2, 3].map((x) => x * 2); // runs 3 times, right now
[5, 3, 8].sort((a, b) => a - b); // runs during sort, right now
```

**Asynchronous** — handed to the runtime, enqueued in the task queue, picked up by the event loop later:

```js
setTimeout(() => console.log("later"), 1000);
fetch("/api").then((res) => console.log(res));
```

The **receiving function** determines sync vs async — not the callback itself. The same arrow function could be sync in one context and async in another.

## Error Handling: Why try/catch Breaks

`try/catch` works for synchronous code. It does **not** work for async callbacks:

```js
try {
  setTimeout(() => {
    throw new Error("boom"); // NOT caught
  }, 0);
} catch (err) {
  // never reached
}
```

The `try/catch` wraps the `setTimeout` call, which succeeds immediately (it just registers a timer). The error is thrown later, in a different call stack frame — the one the event loop creates when it picks the callback off the queue. By then, the `try` block is gone.

### Error-First Convention

Since `try/catch` can't cross async boundaries, Node.js established a pattern — pass the error as the first argument:

```js
readFile("/data.txt", (err, data) => {
  if (err) {
    console.error(err.message);
    return;
  }
  console.log(data);
});
```

Rules:

1. First argument is the error (`null` on success).
2. Remaining arguments are result data.
3. Consumer **must** check `err` before using data.

This is a **convention**, not a language feature. Nothing enforces it. Forget to check `err` and the code silently continues with `null`/`undefined` data — errors vanish instead of surfacing.

## Pros and Cons

| Pros                                    | Cons                                                               |
| --------------------------------------- | ------------------------------------------------------------------ |
| Simple concept — just passing functions | Error handling is manual and fragile                               |
| Universal — any JS version, any runtime | Inversion of control — you trust the receiver to call it correctly |
| Fine for simple one-level async         | Composition is painful (nesting → callback hell)                   |
|                                         | No built-in cancellation or status tracking                        |

## Callbacks in the Wild

| API                               | Sync/Async | Notes                                         |
| --------------------------------- | ---------- | --------------------------------------------- |
| `Array.map/filter/reduce/forEach` | Sync       | Language built-ins, callback runs immediately |
| `setTimeout` / `setInterval`      | Async      | Runtime timer APIs                            |
| `addEventListener`                | Async      | Runtime DOM API, fires on user events         |
| `fs.readFile` (Node.js)           | Async      | Runtime file I/O, error-first pattern         |
| Express route handlers            | Async      | Framework invokes on incoming requests        |

Even after promises and async/await, callbacks remain the underlying mechanism — promises are built on top of them.
