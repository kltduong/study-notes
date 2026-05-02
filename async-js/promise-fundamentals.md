# Promise Fundamentals

**TL;DR:** A promise is a state machine (`pending → fulfilled | rejected`) that settles exactly once — the language-level fix for callback inversion of control. The executor runs synchronously; `.then()` handlers always run asynchronously (as microtasks). Rejections propagate through the chain until handled — missing a handler doesn't swallow the error, it passes it along.

## What Problem Promises Solve

Callbacks have three structural problems: inside-out composition, scattered error handling, and inversion of control (see [callback-problems.md](callback-problems.md)). Promises flip the model — instead of handing your callback to someone else, the async function gives _you_ an object representing the future result. You decide what to do with it.

## Promise States

A promise is a state machine with three states:

```mermaid
graph LR
    P["pending"] -->|"resolve(value)"| F["fulfilled"]
    P -->|"reject(reason)"| R["rejected"]
```

- **Pending** — initial state, async work in progress.
- **Fulfilled** — work succeeded, promise holds a result value.
- **Rejected** — work failed, promise holds a reason (typically an `Error`).

Once a promise moves to fulfilled or rejected, it's **settled**. The critical guarantee: **a promise settles exactly once.** Can't go back to pending, can't settle twice, can't switch from fulfilled to rejected. This is the structural fix for inversion of control — callbacks could be called twice or never; a promise settles once, enforced by the language.

## Creating a Promise

The `Promise` constructor takes an **executor** function that receives `resolve` and `reject`:

```js
const promise = new Promise((resolve, reject) => {
  // async work...
  // resolve(value) on success
  // reject(reason) on failure
});
```

The executor runs **immediately and synchronously** — a common misconception is that creating a promise defers execution. It doesn't. What's deferred is the `.then()` handler reaction.

```js
console.log("before");
const p = new Promise((resolve) => {
  console.log("inside executor"); // runs synchronously
  resolve("done");
});
console.log("after");
// Output: before, inside executor, after
```

## resolve and reject

Two levers that move the promise out of pending:

```js
const success = new Promise((resolve, reject) => {
  resolve(42); // fulfilled with value 42
});

const failure = new Promise((resolve, reject) => {
  reject(new Error("something broke")); // rejected with an Error
});
```

Key behaviors:

- **Only the first call matters.** `resolve()` then `reject()` → fulfilled. The reject is ignored. Vice versa. This is the settle-once guarantee.
- **`resolve` doesn't always mean fulfilled** — resolving with another promise makes the outer promise adopt the inner one's state (relevant for chaining).
- **`reject` should receive an `Error` object** — for stack traces and `instanceof` checks, same reasoning as the error-first convention.

### resolve/reject don't stop execution

`resolve()` and `reject()` are regular function calls, not control flow. Code after them keeps running:

```js
const p = new Promise((resolve, reject) => {
  resolve("done");
  console.log("still runs"); // prints!
  someArray.push("side effect"); // happens!
  reject(new Error("ignored")); // executes, no effect on promise
});
```

The later `resolve`/`reject` calls are harmless _to the promise_, but other code between them runs and can cause side effects. The `return resolve(...)` pattern prevents this:

```js
const p = new Promise((resolve, reject) => {
  if (badInput) return reject(new Error("bad"));
  // nothing below runs if we rejected
  doExpensiveWork();
  resolve(result);
});
```

`return` and `resolve`/`reject` are independent — `return` exits the executor, `resolve`/`reject` settles the promise. The Promise constructor ignores the executor's return value.

## Consuming with `.then()`

`.then()` attaches reactions to a promise:

```js
promise.then(onFulfilled, onRejected);
```

- `onFulfilled` — called if the promise fulfills, receives the value.
- `onRejected` — called if the promise rejects, receives the reason.

```js
const promise = new Promise((resolve) => {
  setTimeout(() => resolve("data loaded"), 1000);
});

promise.then(
  (value) => console.log(value), // "data loaded" — after 1 second
  (reason) => console.error(reason), // not called
);
```

### `.then()` handlers always run asynchronously

Even if the promise is already settled when `.then()` is attached, the handler is enqueued as a **microtask** — never called synchronously:

```js
const p = Promise.resolve("instant");
p.then((v) => console.log(v));
console.log("after .then()");
// Output: "after .then()", "instant"
```

This is a design choice for consistency — you never have to wonder "did this run sync or async?" like with callbacks.

### Enqueuing trigger

Whichever happens second — settling or attaching — triggers the microtask enqueue:

- **`resolve()` before `.then()`:** `.then()` sees the promise is already settled → enqueues the handler immediately.
- **`.then()` before `resolve()`:** `.then()` registers the handler and waits → `resolve()` settles the promise, sees a registered handler → enqueues it.

Either way, the handler runs after the stack drains.

## Rejection Propagation

When `.then()` has no `onRejected` handler, the rejection **propagates** — it passes through to the promise `.then()` returns and keeps flowing until something handles it:

```js
Promise.reject(new Error("oops"))
  .then((v) => console.log(v)) // no onRejected → passes through
  .then((v) => console.log(v)) // no onRejected → passes through
  .then(null, (e) => console.log(e.message)); // caught: "oops"
```

If nothing handles it, the runtime raises an **unhandled promise rejection** warning (browsers log it, Node.js can crash). This is a key improvement over callbacks, where forgetting `if (err)` meant errors silently vanished.

**Missing handler vs empty handler:**

```js
// A — missing handler: rejection propagates
p.then((val) => console.log(val));

// B — empty handler: rejection is swallowed (caught and ignored)
p.then(
  (val) => console.log(val),
  () => {},
);
```

Missing handler = propagation. Empty handler = swallowing.

## How Promises Fix Callback Problems

| Callback problem                                     | Promise solution                                 |
| ---------------------------------------------------- | ------------------------------------------------ |
| Inversion of control — trust the receiver            | Promise settles once, guaranteed by the language |
| Scattered error handling — `if (err)` at every level | Rejections propagate; handle in one place        |
| Inside-out composition — nested callbacks            | `.then()` returns a new promise → flat chaining  |
