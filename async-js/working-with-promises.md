# Working with Promises

**TL;DR:** Promisification wraps callback-based functions in a `new Promise`, mapping the error-first callback to `reject`/`resolve`. `.then()` always returns a new promise synchronously — the handler's return value settles it later. Returning a promise from a handler makes the chain wait. Chaining (sequential) and branching (independent) are different wiring patterns on the same mechanic.

## Rewriting Callbacks as Promises

A callback-based function traps its result inside the callback — the caller hands over control and hopes for the best. The promise version returns an object the caller owns:

```js
// Callback version — caller gives up control
function fetchUser(id, callback) {
  setTimeout(() => {
    if (!id) return callback(new Error("No ID"));
    callback(null, { id, name: "Alice" });
  }, 500);
}

// Promise version — caller gets an object back
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (!id) return reject(new Error("No ID"));
      resolve({ id, name: "Alice" });
    }, 500);
  });
}
```

What changes structurally:

- **No `callback` parameter.** The function returns a promise instead. The caller decides what to do with the result.
- **Error handling is unified.** `reject()` replaces `callback(err)`. The consumer handles it via `onRejected` or lets it propagate.
- **The return value is useful.** A callback-based function returns `undefined`. A promise-returning function gives you an object you can store, pass around, chain, or combine.

## Promisification

Wrapping an existing callback-based function in a promise — useful for APIs you can't rewrite (Node.js `fs`, older libraries):

```js
function promisifiedReadFile(path) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, "utf8", (err, data) => {
      if (err) return reject(err);
      resolve(data);
    });
  });
}
```

The pattern is mechanical:

1. Return a `new Promise`.
2. Call the original function inside the executor.
3. In the callback: `reject(err)` if error, `resolve(data)` if success.

A generic utility:

```js
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) return reject(err);
        resolve(result);
      });
    });
  };
}

const readFile = promisify(fs.readFile);
readFile("data.txt", "utf8").then((data) => console.log(data));
```

Node.js ships `util.promisify` that does this with extra edge-case handling.

**Key assumption:** promisification relies on the **error-first callback convention** — `(err, result)`. The wrapper needs to know the first argument is the error and the second is the data. A library with a different callback shape (arguments flipped, no error argument, multiple results) breaks the generic wrapper.

## Promise Chaining

`.then()` returns a new promise, so calls can be chained flat instead of nested:

```js
// Chained — flat, sequential
fetchUser(1)
  .then((user) => fetchPosts(user.id))
  .then((posts) => fetchComments(posts[0].id))
  .then((comments) => console.log(comments));

// Callback equivalent — nested
fetchUser(1, (err, user) => {
  fetchPosts(user.id, (err, posts) => {
    fetchComments(posts[0].id, (err, comments) => {
      console.log(comments);
    });
  });
});
```

### How chaining works

The handler's return value controls what the `.then()` promise settles to:

| Handler returns       | `.then()` promise becomes                            |
| --------------------- | ---------------------------------------------------- |
| A regular value       | Fulfilled with that value                            |
| A promise             | Adopts that promise's state (waits for it to settle) |
| Throws an error       | Rejected with that error                             |
| Nothing (`undefined`) | Fulfilled with `undefined`                           |

The second row is the key — returning a promise makes the chain **wait** for it to settle before the next handler fires. This is what makes sequential async operations flat.

See [promise-fundamentals.md](promise-fundamentals.md) for the two-phase mechanic: `.then()` creates its promise synchronously at chain construction time; the handler's return value settles that already-existing promise later.

### Chaining vs Branching

```js
// Branching — two independent reactions to the same promise
const p = fetchUser(1);
p.then((user) => fetchPosts(user.id));
p.then((user) => fetchProfile(user.id));
// Neither waits for the other — they run independently

// Chaining — sequential, each step waits for the previous
fetchUser(1)
  .then((user) => fetchPosts(user.id))
  .then((posts) => renderPosts(posts));
// renderPosts waits for fetchPosts to finish
```

Branching isn't wrong — it's useful for parallel independent work from the same starting point. But the results don't flow from one step to the next.

### Error propagation through chains

Rejections skip `onFulfilled` handlers and propagate until they hit an `onRejected`:

```js
fetchUser(1)
  .then((user) => fetchPosts(user.id)) // skipped if fetchUser rejects
  .then((posts) => renderPosts(posts)) // skipped if fetchPosts rejects
  .then(null, (err) => console.error(err)); // catches any rejection above
```

Single error handler at the end — the promise equivalent of `try/catch` around the whole sequence. Compare with callbacks where every level needs its own `if (err)` check.
