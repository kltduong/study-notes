# Async/Await

**TL;DR:** `async` functions always return a promise. `await` suspends the function at that point and yields control back to the caller — everything before the first `await` runs synchronously. Sequential vs parallel execution depends on _where_ you `await`, not whether you use it.

## How Async Functions Execute

An `async` function runs synchronously up to its first `await`, then suspends. Control returns to the caller, and the rest of the function resumes later as a microtask when the awaited value settles.

```js
async function getUser() {
  console.log("A"); // sync — runs immediately
  const resp = await fetch("/api/user"); // suspends here
  console.log("B"); // resumes as microtask after fetch settles
  return resp.json();
}

console.log("X");
const p = getUser(); // "X" then "A" — sync portion runs
console.log("Y"); // "Y" — getUser is suspended, caller continues
// later: "B"
```

Output: `X, A, Y, B`. At the moment `"Y"` logs, `p` is a pending promise.

## Await and Suspension

`await expr` does two things:

1. Evaluates `expr` (if it's not already a promise, wraps it in `Promise.resolve()`).
2. Suspends the async function. The function's continuation is scheduled as a microtask for when the promise settles.

The suspension is _local to that function_. Other code — including other async functions already in flight — keeps running.

## Error Handling with Try/Catch

`await` lets you use synchronous-style `try/catch` for async errors:

```js
async function init() {
  try {
    const config = await loadConfig();
    console.log(config.theme);
  } catch (e) {
    console.log("caught:", e.message);
  }
}
```

If the awaited promise rejects, the rejection is thrown at the `await` site and caught by the surrounding `try/catch`. This is the main ergonomic win over `.then()`/`.catch()` chains.

**Fetch gotcha still applies:** `fetch` fulfills on HTTP errors (404, 500). A 404 response won't trigger the `catch` block — you still need to check `res.ok`.

## Sequential vs Parallel

### Sequential (slow)

`await` inside a loop makes each iteration wait for the previous one:

```js
async function fetchAll(ids) {
  const results = [];
  for (const id of ids) {
    const res = await fetch(`/api/item/${id}`); // waits here
    const data = await res.json(); // then here
    results.push(data); // next iteration starts after
  }
  return results;
}
```

N requests × latency each. Each fetch starts only after the previous one completes.

### Parallel (fast)

Fire all requests without awaiting in the loop, then collect with `Promise.all`:

```js
// Using .then() chains
async function fetchAll(ids) {
  const promises = [];
  for (const id of ids) {
    const p = fetch(`/api/item/${id}`).then((res) => res.json());
    promises.push(p);
  }
  return Promise.all(promises);
}

// Using an async helper
async function fetchAll(ids) {
  const promises = [];
  async function fetchOne(id) {
    const res = await fetch(`/api/item/${id}`);
    return res.json();
  }
  for (const id of ids) {
    promises.push(fetchOne(id)); // no await — fires and moves on
  }
  return Promise.all(promises);
}
```

Both approaches: all fetches start immediately, run concurrently, and `Promise.all` preserves input order regardless of completion order.

### Why the helper version is still parallel

Calling `fetchOne(id)` without `await` returns a promise immediately. The `await` _inside_ `fetchOne` only suspends that particular invocation — it doesn't block the loop. Each `fetchOne` call is an independent chain of microtask continuations that progresses on its own. They interleave freely:

```
fetchOne("a"): fetch starts → suspends
fetchOne("b"): fetch starts → suspends
fetchOne("c"): fetch starts → suspends
── call stack empty, event loop takes over ──
fetchOne("b"): fetch resolves → json starts → suspends
fetchOne("a"): fetch resolves → json starts → suspends
fetchOne("c"): fetch resolves → json starts → suspends
fetchOne("a"): json resolves → done
fetchOne("c"): json resolves → done
fetchOne("b"): json resolves → done
```

The rule: parallelism is controlled by where you `await` in the _calling_ code, not inside the helper.

## `return await` vs `return`

```js
async function fetchOne(id) {
  const res = await fetch(`/api/item/${id}`);
  return await res.json(); // unwraps then re-wraps
}

async function fetchOne(id) {
  const res = await fetch(`/api/item/${id}`);
  return res.json(); // passes promise through directly
}
```

Functionally identical in most cases. `return await` unwraps the promise then the async function re-wraps the value — one extra microtask tick, negligible.

**The one case it matters: `try/catch`.** With `return promise` (no await), the function has already returned by the time the promise rejects, so the `catch` block never fires. With `return await promise`, the await happens inside the function, so a rejection is thrown at the `await` site and caught normally.

```js
async function safe(id) {
  try {
    const res = await fetch(`/api/item/${id}`);
    return await res.json(); // rejection caught below
  } catch (e) {
    return fallback;
  }
}
```

Outside of `try/catch`, prefer `return promise` — simpler, one fewer tick.

## Top-Level Await

ES modules support `await` at the top level — no wrapping async function needed. The module's evaluation suspends until the awaited value settles. Importing modules wait for it.

```js
// config.mjs
const res = await fetch("/config");
export const config = await res.json();
```

Only works in ES modules (`.mjs` or `type: "module"`), not in scripts or CommonJS.
