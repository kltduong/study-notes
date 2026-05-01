# Testing Callbacks

**TL;DR:** Test runners don't know when async work finishes unless you tell them. Mocha's `done` callback is the signal — accept it as a parameter to enter async mode, call it when the work is complete. Without it, async tests silently pass. Chai assertions work by throwing on failure, which means a failed assertion inside an async callback can go uncaught and produce a misleading timeout instead of the real error.

## The Core Problem

Test runners execute your test function and consider it done when the function returns. With callbacks, the result arrives _later_ — on a future event loop tick. If you don't tell the runner to wait, it declares success before the callback fires:

```js
it("should return user data", () => {
  getUser(1, (err, user) => {
    assert.equal(user.name, "Alice"); // runs AFTER Mocha moved on
  });
  // returns here — Mocha thinks it passed
});
```

The test passes even if the assertion would fail. A false positive — the most dangerous kind of test bug.

This is the same "try/catch can't cross async boundaries" problem from [callbacks.md](callbacks.md), applied to testing.

## How Mocha Invokes Tests

Mocha works in two phases: **registration** then **execution**.

### Phase 1 — Registration

When your test file loads, `it()` doesn't run your test function. It stores it:

```js
// Simplified Mocha internals
function it(description, testFn) {
  testSuite.push({
    description: description,
    fn: testFn, // your function — stored as a reference, NOT called
  });
}
```

So when you write:

```js
it("should do something", function (done) {
  this.timeout(5000);
  getUser(1, (err, user) => {
    assert.equal(user.name, "Alice");
    done();
  });
});
```

Mocha just stores `{ description: "should do something", fn: <your function> }`. Nothing executes yet. All `it()` calls across all test files are collected first.

### Phase 2 — Execution

Once all tests are registered, Mocha runs them one by one. For each test, it checks `testFn.length` — the number of declared formal parameters — to decide sync vs async mode:

```js
const testFn = test.fn;
const testContext = { timeout: ..., slow: ..., retries: ... };

if (testFn.length > 0) {
  // ≥1 parameter declared → async mode
  const done = createDoneCallback();
  testFn.call(testContext, done);
  // wait for done() to be called...
} else {
  // 0 parameters → sync mode
  testFn.call(testContext);
  // returned without throwing → pass
}
```

The `.call(testContext, done)` does two things:

- **First argument** → becomes `this` inside the function (Mocha's context object with `timeout()`, `slow()`, etc.)
- **Second argument** → becomes the `done` parameter

`testFn.length` is a built-in property on every function — it returns the count of declared parameters. That's the entire detection mechanism. Mocha doesn't parse your code or look for the word "done":

```js
((done) => {})
  .length(
    // 1 → async mode
    () => {},
  )
  .length(
    // 0 → sync mode
    function (done) {},
  )
  .length(
    // 1 → async mode
    function () {},
  ).length; // 0 → sync mode
```

### Why arrow functions break `this` but not `done`

Both function styles receive `done` correctly — it's a regular argument. The difference is only `this`:

```js
it("test", function (done) {
  // this === testContext  ✅  .call() sets it
  // done === doneCallback ✅  passed as argument
});

it("test", (done) => {
  // this === <enclosing scope's this>  ❌  arrow ignores .call()
  // done === doneCallback              ✅  arguments still work
});
```

Arrow functions are fine for most tests. They only break when you need `this.timeout()`, `this.slow()`, or `this.retries()` — methods on Mocha's context object that `this` must point to.

## The `done` Callback

If your test function accepts a parameter, Mocha switches to async mode and **waits** for you to call it:

```js
it("should return user data", (done) => {
  getUser(1, (err, user) => {
    if (err) return done(err);
    assert.equal(user.name, "Alice");
    done(); // signals Mocha: "finished, check results"
  });
});
```

The mechanics:

1. Mocha sees one parameter → async mode.
2. Starts a timeout clock (default 2000ms).
3. Test function runs, registers the async operation, returns.
4. Mocha waits — does **not** mark the test as passed.
5. `done()` is called → Mocha finalizes the test.
6. `done()` never called within timeout → test fails with a timeout error.

## Two Ways to Signal Failure

### Forwarding unexpected errors

When the function under test produces an error you didn't expect, pass it straight to `done`:

```js
it("should return user data", (done) => {
  getUser(1, (err, user) => {
    if (err) return done(err); // fail immediately with the real error
    assert.equal(user.name, "Alice");
    done();
  });
});
```

`done(err)` tells Mocha: "this test is finished and it failed — here's why." Without it, the test would either throw inside the callback (uncaught) or continue with `undefined` data and fail on an assertion — both leading to a useless timeout message instead of the real cause.

### Catching assertion failures

Chai assertions throw an `AssertionError` on failure. Inside an async callback, that throw goes uncaught — Mocha isn't wrapping that callback in a try/catch. `done()` is never reached, and you get a timeout:

```js
getUser(1, (err, user) => {
  assert.isNull(err); // throws AssertionError if err is truthy
  assert.equal(user.name, "Alice"); // never reached
  done(); // never reached
});
// Result: timeout — hides the real assertion failure
```

Fix with try/catch:

```js
it("should fetch user", (done) => {
  getUser(1, (err, user) => {
    try {
      assert.isNull(err);
      assert.equal(user.name, "Alice");
      done();
    } catch (e) {
      done(e); // hands the AssertionError to Mocha directly
    }
  });
});
```

Now Mocha reports "expected Error to be null" instead of a generic timeout.

## Testing Expected Errors

When the error _is_ the expected behavior, assert on it and call `done()` with no arguments — the test passed:

```js
it("should reject invalid IDs", (done) => {
  getUser(-1, (err, user) => {
    assert.instanceOf(err, Error);
    assert.include(err.message, "not found");
    done(); // no argument — test passed, error was expected
  });
});
```

## `done` Called Twice

Mocha throws if `done()` is called more than once. This catches a real bug: callbacks that fire multiple times.

```js
function getUser(id, callback) {
  const xhr = new XMLHttpRequest();
  xhr.open("GET", `/api/users/${id}`);
  xhr.onload = function () {
    if (xhr.status === 200) {
      callback(null, JSON.parse(xhr.responseText));
    }
    // bug: missing return/else — falls through on 200
    callback(new Error(`Status: ${xhr.status}`));
  };
  xhr.send();
}
```

On a 200 response, the callback fires twice — once with data, once with a bogus error. In production this causes subtle bugs (double renders, phantom errors). In a Mocha test, `done()` is called twice and Mocha flags it immediately.

This connects to [inversion of control](callback-problems.md): you hand your callback to someone else's function and trust it to call it exactly once. Nothing in the language enforces that. Mocha's double-call detection is a safeguard the callback pattern itself lacks. Promises solve this structurally — a promise can only settle once.

## Mocha Timeouts

Default timeout is 2000ms. Configure with `this.timeout()`:

```js
it("should handle slow operations", function (done) {
  this.timeout(5000);
  slowOperation((err, result) => {
    if (err) return done(err);
    assert.isOk(result);
    done();
  });
});
```

Requires a regular `function`, not an arrow function — see [How Mocha Invokes Tests](#how-mocha-invokes-tests) for why.

## Chai Assertion Styles

Three ways to write the same assertion:

```js
assert.equal(user.name, "Alice"); // assert style
expect(user.name).to.equal("Alice"); // expect style (BDD)
user.name.should.equal("Alice"); // should style (extends Object.prototype)
```

All throw `AssertionError` on failure. `expect` is most common in modern codebases — reads naturally without modifying prototypes.
