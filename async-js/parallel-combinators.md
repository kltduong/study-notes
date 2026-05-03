# Parallel Combinators: Promise.all & Promise.allSettled

**TL;DR:** `Promise.all` waits for every promise to fulfill (short-circuits on first rejection). `Promise.allSettled` waits for every promise to settle and always fulfills — individual outcomes are captured as `{ status, value/reason }` objects. Both preserve input order, not completion order. Neither cancels in-flight promises.

## The Problem: Waiting for Multiple Independent Promises

Chaining serializes operations — each waits for the previous one. When operations are independent (fetch profile, posts, notifications), you want them in parallel. Three 200ms requests chained = 600ms. In parallel = ~200ms.

```js
// Sequential — unnecessarily slow
fetchProfile(userId).then((profile) => fetchPosts(userId).then((posts) => render(profile, posts)));

// Parallel — fire all, wait for slowest
Promise.all([fetchProfile(userId), fetchPosts(userId)]).then(([profile, posts]) => render(profile, posts));
```

The promises are created (and their executors run) at call time — `Promise.all` doesn't start them, it just watches them.

## `Promise.all`

Takes an iterable of promises. Returns a single promise that:

- **Fulfills** when all input promises fulfill — with an array of values in input order.
- **Rejects** when any input promise rejects — with the first rejection reason.

```js
Promise.all([fetch("/api/users"), fetch("/api/posts"), fetch("/api/comments")])
  .then(([users, posts, comments]) => render(users, posts, comments))
  .catch((err) => showError(err));
```

### Key Behaviors

**Order preserved.** Results match input position, not completion order. `[slow, fast]` → `[slowResult, fastResult]`.

**Non-promise values are fine.** Treated as already-fulfilled via `Promise.resolve()`:

```js
Promise.all([1, Promise.resolve(2), 3]); // fulfills with [1, 2, 3]
```

**Short-circuits on first rejection.** Rejects immediately — doesn't wait for remaining promises. But it **doesn't cancel** them. They keep running; results are discarded. JS has no built-in promise cancellation.

```js
Promise.all([
  Promise.resolve("A"),
  Promise.reject(new Error("B failed")),
  Promise.resolve("C"), // still runs, result ignored
]).catch((err) => console.log(err.message)); // "B failed"
```

**Empty array** → fulfills immediately with `[]`.

### From-Scratch Implementation

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let settled = 0;
    const items = Array.from(promises);

    if (items.length === 0) return resolve([]);

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = value; // preserve order, not push
          settled++;
          if (settled === items.length) resolve(results);
        },
        (reason) => reject(reason), // first rejection wins
      );
    });
  });
}
```

Implementation details:

- **`Promise.resolve(item)`** — normalizes non-promise values so `.then()` works on anything (see [fetch-rejections.md](fetch-rejections.md) for `Promise.resolve` behavior).
- **`results[index]`** not `results.push()` — push gives completion order, index assignment gives input order.
- **`settled` counter** not `results.length` — sparse arrays have misleading `.length`. If index 2 resolves first, `results.length` is 3 but only one slot is filled.
- Multiple `reject()` calls are harmless — settle-once guarantee.

## `Promise.allSettled`

Same input as `Promise.all`. Returns a promise that **always fulfills** when all input promises settle. Result is an array of outcome objects:

```js
Promise.allSettled([Promise.resolve("ok"), Promise.reject(new Error("fail")), Promise.resolve(42)]).then((results) =>
  console.log(results),
);
// [
//   { status: "fulfilled", value: "ok" },
//   { status: "rejected", reason: Error("fail") },
//   { status: "fulfilled", value: 42 }
// ]
```

Each outcome object:

- `status` — `"fulfilled"` or `"rejected"` (string).
- `value` — present only when fulfilled.
- `reason` — present only when rejected.

### From-Scratch Implementation

Simpler than `Promise.all` — no short-circuit logic:

```js
function promiseAllSettled(promises) {
  return new Promise((resolve) => {
    const results = [];
    let settled = 0;
    const items = Array.from(promises);

    if (items.length === 0) return resolve([]);

    items.forEach((item, index) => {
      Promise.resolve(item).then(
        (value) => {
          results[index] = { status: "fulfilled", value };
          settled++;
          if (settled === items.length) resolve(results);
        },
        (reason) => {
          results[index] = { status: "rejected", reason };
          settled++;
          if (settled === items.length) resolve(results);
        },
      );
    });
  });
}
```

The outer promise only ever calls `resolve()` — both fulfillment and rejection of individual promises become data in the results array.

### Building `allSettled` from `Promise.all`

Catch each promise individually so none can reject, then `Promise.all` on an array that never rejects:

```js
function allSettled(promises) {
  const wrapped = Array.from(promises).map((p) =>
    Promise.resolve(p).then(
      (value) => ({ status: "fulfilled", value }),
      (reason) => ({ status: "rejected", reason }),
    ),
  );
  return Promise.all(wrapped);
}
```

Each `.then(onFulfilled, onRejected)` converts rejections into fulfilled outcome objects — a practical example of `.catch()` recovery (see [fetch-rejections.md](fetch-rejections.md)).

## `Promise.all` vs `Promise.allSettled`

|                   | `Promise.all`                    | `Promise.allSettled`              |
| ----------------- | -------------------------------- | --------------------------------- |
| **Fulfills when** | All fulfill                      | All settle                        |
| **Rejects when**  | Any rejects (short-circuits)     | Never                             |
| **Result shape**  | Array of values                  | Array of `{status, value/reason}` |
| **Use case**      | All-or-nothing — failure = abort | Best-effort — report each outcome |

**Use `Promise.all`** when operations are interdependent — if one fails, the whole batch is useless (e.g., fetching all parts of a page layout).

**Use `Promise.allSettled`** when operations are independent — you want to know what succeeded and what failed without one blocking the others (e.g., sending analytics to multiple providers).
