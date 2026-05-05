# Fetch & Rejection Handling

**TL;DR:** `fetch()` is the browser's promise-based HTTP API. It only rejects on network failures — HTTP errors (404, 500) fulfill normally, so you must check `response.ok` yourself. `.catch()` is sugar for `.then(null, onRejected)` and **recovers** the chain by default. `.finally()` is transparent — runs cleanup without touching the settlement. `Promise.resolve()`/`Promise.reject()` create pre-settled promises for normalizing interfaces and starting chains.

## Fetch: Two-Phase Response

`fetch()` returns a promise that resolves to a `Response` object in two phases:

1. **Headers arrive** → `fetch()` promise fulfills with a `Response`. You have status, headers, content type.
2. **Body is read** → call `.json()`, `.text()`, `.blob()` on the Response. Returns another promise because the body may still be streaming.

```js
fetch("https://api.example.com/users/1")
  .then((response) => response.json()) // phase 2: parse body
  .then((data) => console.log(data)); // actual data
```

Two `.then()` calls isn't ceremony — it reflects the network reality that headers and body arrive separately.

## HTTP Errors Don't Reject

`fetch()` only rejects on **network failures** — DNS errors, no internet, CORS blocked. An HTTP 404 or 500 **fulfills** the promise. The server responded, so from fetch's perspective the request succeeded.

```js
fetch("https://api.example.com/nonexistent").then((response) => {
  console.log(response.status); // 404
  console.log(response.ok); // false — but promise fulfilled
});
```

`response.ok` is `true` for 200–299, `false` otherwise. To turn HTTP errors into rejections:

```js
fetch("https://api.example.com/users/1").then((response) => {
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
});
```

Throwing inside `.then()` rejects the promise it returned — the error propagates down the chain like any other rejection.

## `.catch()` — Sugar for `.then(null, onRejected)`

```js
// Identical:
promise.then(null, (err) => handle(err));
promise.catch((err) => handle(err));
```

Idiomatic pattern — `.catch()` at the end of a chain catches rejections from any step above:

```js
fetch("/api/users/1")
  .then((response) => {
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  })
  .then((data) => renderUser(data))
  .catch((err) => showError(err));
```

### `.catch()` recovers the chain

`.catch()` returns a new promise. If the handler returns normally (doesn't throw), that promise **fulfills** — the chain recovers as if nothing went wrong:

```js
Promise.reject(new Error("fail"))
  .catch((err) => "recovered")
  .then((val) => console.log(val)); // "recovered"
```

Downstream `.then()` handlers see a fulfilled promise and run their `onFulfilled`. To catch, log, and keep the rejection flowing — re-throw:

```js
.catch((err) => {
  console.error(err);
  throw err; // re-reject
})
```

## `.finally()` — Transparent Cleanup

Runs whether the promise fulfills or rejects. Designed for side effects (hide spinners, close connections), not value transformation.

```js
showSpinner();
fetch("/api/users/1")
  .then((response) => response.json())
  .then((data) => renderUser(data))
  .catch((err) => showError(err))
  .finally(() => hideSpinner());
```

Key behaviors:

- **No arguments** — handler doesn't receive the value or reason.
- **Return value ignored** — whatever the handler returns is discarded.
- **Transparent** — passes through the original settlement (fulfillment or rejection) as if `.finally()` wasn't there.
- **One exception:** if the handler **throws**, the chain rejects with that error and the original settlement is lost.

```js
Promise.resolve("hello")
  .finally(() => "ignored") // "hello" passes through
  .finally(() => {
    throw new Error("oops");
  }) // overrides → rejected
  .finally(() => "also ignored") // rejection passes through (transparent both ways)
  .then(null, (err) => err.message); // "oops"
```

## Throw Behavior Is Universal

Throwing inside any handler — `.then()`, `.catch()`, or `.finally()` — rejects the promise that call returned. The difference is what happens on **normal return**:

| Handler returns normally | Effect on chain                                  |
| ------------------------ | ------------------------------------------------ |
| `.then()`                | Fulfills with the returned value (replaces)      |
| `.catch()`               | Fulfills with the returned value (recovers)      |
| `.finally()`             | Passes through original settlement (transparent) |

## `Promise.resolve()` and `Promise.reject()`

Static methods that create already-settled promises:

```js
const fulfilled = Promise.resolve(42);
const rejected = Promise.reject(new Error("nope"));
```

### When to use vs `new Promise`

**Have the value synchronously** → `Promise.resolve()` / `Promise.reject()`. No async work, no executor needed.
**Need async work to produce the value** → `new Promise`. The executor gives you `resolve`/`reject` to call later when the work finishes.

```js
// Value in hand — Promise.resolve
if (cache.has(id)) return Promise.resolve(cache.get(id));

// Need to wait — new Promise
function delay(ms) {
  return new Promise((resolve) => setTimeout(() => resolve(), ms));
}
```

You _could_ write `new Promise((resolve) => resolve(cachedValue))` — it works, but the executor runs synchronously, calls resolve immediately, and exits. `Promise.resolve()` is the shorthand for that exact pattern.

### Use cases

**Normalizing sync/async interfaces** — always return a promise so the consumer doesn't branch:

```js
function getUser(id) {
  if (cache.has(id)) return Promise.resolve(cache.get(id));
  return fetch(`/api/users/${id}`).then((r) => r.json());
}
// Consumer always: getUser(1).then(...)
```

**Starting a chain** when there's no initial promise:

```js
Promise.resolve()
  .then(() => step1())
  .then(() => step2());
```

### `Promise.resolve()` adopts thenables

Passing a promise (or any object with `.then()`) to `Promise.resolve()` doesn't wrap it — it **adopts** it:

```js
const original = fetch("/api/data");
const adopted = Promise.resolve(original);
console.log(adopted === original); // true
```

This means `Promise.resolve(p)` and `new Promise((resolve) => resolve(p))` are **not** identical when `p` is a promise. `Promise.resolve` returns the same object; `new Promise` always creates a new one (different identity, extra microtick for adoption). For plain values they're interchangeable.

`Promise.reject()` does **not** adopt — it always wraps. `Promise.reject(somePromise)` gives a rejected promise whose reason is the promise object itself.
