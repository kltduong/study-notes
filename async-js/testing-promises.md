# Testing Promises

**TL;DR:** Return the promise from your test function — Mocha waits for it to settle and catches assertion errors automatically. Forget the `return` and you get a false positive (same trap as forgetting `done` with callbacks). Never mix `done` and a returned promise — Mocha rejects the ambiguity. When testing expected rejections, guard against unexpected fulfillment or your assertions silently never run.

## The Promise Advantage: Return It

Mocha has a second async detection mechanism beyond `done`: **if your test function returns a promise, Mocha waits for it to settle.**

```js
it("should fetch user data", () => {
  return getUser(1).then((user) => {
    assert.equal(user.name, "Alice");
  });
});
```

No `done` parameter, no try/catch wrapping. Mocha sees the returned promise and:

- **Fulfills** → test passes
- **Rejects** → test fails with the rejection reason

This solves both problems that plagued [callback testing](testing-callbacks.md):

1. **Signaling completion** — promise settlement is the signal. Nothing to forget to call.
2. **Catching assertion errors** — Chai throws inside `.then()` become rejections, which Mocha catches with the real error message. No manual try/catch + `done(e)` needed.

## The Critical Mistake: Forgetting `return`

```js
it("should fetch user data", () => {
  getUser(1).then((user) => {
    assert.equal(user.name, "WRONG NAME");
  });
  // no return — function returns undefined
});
```

Mocha sees `undefined` (not a promise), treats it as a sync test, and **passes immediately**. The `.then()` handler fires later on a microtask, the assertion throws, nobody catches it. False positive — same result as forgetting `done`, different mechanism.

This is actually _easier_ to get wrong than `done`. A missing `done` parameter is visible in the function signature. A missing `return` is invisible.

## `done` Still Works — But Don't Mix

You _can_ use `done` with promise-based code:

```js
it("should fetch user data", (done) => {
  getUser(1)
    .then((user) => {
      assert.equal(user.name, "Alice");
      done();
    })
    .catch(done);
});
```

But this is strictly worse — you're back to manual signaling. And there's a hard rule: **never combine `done` with a returned promise.** Mocha throws "Resolution method is overspecified":

```js
// ❌ Mocha error
it("bad test", (done) => {
  return getUser(1).then((user) => {
    assert.equal(user.name, "Alice");
    done();
  });
});
```

Two completion signals are ambiguous — they could disagree (promise settles but `done` never called, or vice versa). Rather than pick one and risk subtle bugs, Mocha rejects it outright.

## Testing Expected Rejections

When you _expect_ a promise to reject, a naive `.catch()` creates a false positive:

```js
// ❌ Passes if fetchUser fulfills — .catch() is skipped, no assertions run
it("should reject invalid IDs", () => {
  return fetchUser(-1).catch((err) => {
    assert.include(err.message, "not found");
  });
});
```

If `fetchUser(-1)` fulfills instead of rejecting, `.catch()` never runs, the promise fulfills, and Mocha sees success. Your assertions were silently skipped.

The fix — **guard against unexpected fulfillment** using the two-argument `.then()`:

```js
it("should reject invalid IDs", () => {
  return fetchUser(-1).then(
    () => {
      throw new Error("Expected rejection, got fulfillment");
    },
    (err) => {
      assert.instanceOf(err, Error);
      assert.include(err.message, "not found");
    },
  );
});
```

This uses `.then(onFulfilled, onRejected)` — not `.then().catch()`. With the chained form, a throw in `onFulfilled` would be caught by `.catch()`, potentially masking the failure.

## Async/Await in Tests

An `async` function always returns a promise, so Mocha's promise detection kicks in automatically:

```js
it("should fetch user data", async () => {
  const user = await getUser(1);
  assert.equal(user.name, "Alice");
});
```

No `return` to forget. Assertions throw synchronously within the function body, and that rejection propagates to Mocha. The cleanest pattern — but the mechanics of _why_ it works are covered in [the async/await chunk, not here].

## Timeouts

Same mechanism as callbacks — default 2000ms. If the promise doesn't settle in time, Mocha fails the test:

```js
it("should handle slow API", function () {
  this.timeout(5000);
  return slowFetch("/api/data").then((data) => {
    assert.isOk(data);
  });
});
```

Still needs a regular `function` (not arrow) when you need `this.timeout()` — see [testing-callbacks.md](testing-callbacks.md) for why.

## Multiple Assertions

All assertions in a single `.then()` is the simplest approach:

```js
it("should return complete user", () => {
  return getUser(1).then((user) => {
    assert.property(user, "name");
    assert.property(user, "email");
    assert.equal(user.name, "Alice");
  });
});
```

Multiple `.then()` blocks matter when you're testing a **chain of async operations** — each step depends on the previous:

```js
it("should create and then fetch user", () => {
  return createUser({ name: "Bob" })
    .then((created) => {
      assert.property(created, "id");
      return getUser(created.id);
    })
    .then((fetched) => {
      assert.equal(fetched.name, "Bob");
    });
});
```

Each `.then()` represents a real async step, not just grouping assertions.

## The Evolution: Callbacks → Promises → Async/Await

| Approach        | Completion signal                | Assertion errors caught?          | Gotcha                           |
| --------------- | -------------------------------- | --------------------------------- | -------------------------------- |
| `done` callback | Manual `done()` call             | Only with try/catch + `done(e)`   | Forget `done` → false positive   |
| Return promise  | Promise settlement               | Automatically (throw → rejection) | Forget `return` → false positive |
| async/await     | Implicit (async returns promise) | Automatically                     | Cleanest; nothing to forget      |

Callbacks required managing both signaling and error catching manually. Promises automate error catching but require a `return`. Async/await automates both.
