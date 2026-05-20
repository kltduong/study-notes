# Functions as Values — Draft

## Plan (teaching order)

- [x] First-class axiom — what "first-class" means structurally
- [x] Passing functions — higher-order function definition, callback pattern
- [x] Returning functions — factory pattern, closures as the enabling mechanism
- [ ] Function identity & reference semantics — equality, aliasing, mutation implications
- [ ] Bridge to the course — what this enables (the rest of the course derives from here)

---

## First-class axiom

**TL;DR:** "First-class" is not a feature — it's a classification. A value is first-class if it has the same rights as every other value in the language. In JS, functions are objects, so they get object rights automatically.

### The axiom set

Two facts; everything else follows:

1. **Functions are objects.** Specifically, callable objects — they have an internal `[[Call]]` slot that makes `()` work, but structurally they're heap-allocated objects with properties, a prototype chain, identity.
2. **JS has one value-passing rule.** Assignment, argument passing, return — all copy the slot content. For objects (including functions), the slot holds a reference.

From (1) + (2), derive:

| Operation | Works for objects | Works for functions | Why |
|-----------|:-:|:-:|-----|
| Assign to variable | ✓ | ✓ | Slot holds reference |
| Store in array/object | ✓ | ✓ | Same — reference into property slot |
| Pass as argument | ✓ | ✓ | Copies reference into parameter slot |
| Return from function | ✓ | ✓ | Copies reference into caller's slot |
| Compare with `===` | ✓ | ✓ | Compares references (identity) |
| Add properties | ✓ | ✓ | Object → has property map |

No special "function-passing" mechanism exists. It's just the object rules you already know.

### Python comparison

Python is identical here — functions are first-class objects, `def` creates a callable object, assignment copies a reference. The parallel is exact. The difference shows up later: Python's `lambda` is expression-limited (single expression, no statements), while JS arrow functions are full-bodied. This matters for HOF ergonomics — JS can inline complex logic where Python often needs a named `def`.

### Teaser reveal

```js
function greet(name) { return `hi ${name}`; }   // L1 — creates function object on heap
const fns = [greet, greet, greet];               // L2 — three slots, each holding same reference
fns[1] = function(name) { return `bye ${name}`; }; // L3 — slot 1 now points to NEW function object
console.log(fns[0]("A"));                        // L4 — "hi A" (slot 0 still → original)
console.log(fns[1]("B"));                        // L5 — "bye B" (slot 1 → new object)
console.log(greet("C"));                         // L6 — "hi C" (greet binding unchanged)
```

Key insight: `fns[1] = ...` overwrites the *reference in slot 1*. It doesn't mutate the function object, doesn't affect other references to the same object, doesn't touch the `greet` binding. Same semantics as `arr[1] = newObj` for any object.

---

## Passing functions (higher-order functions & callbacks)

**Definition:** A higher-order function (HOF) is a function that takes a function as an argument, returns a function, or both. That's it — no magic, just a function whose parameter slot happens to hold a reference to a callable object.

The term "callback" is a usage pattern, not a language mechanism: a function you pass *to be called later* by someone else. The mechanism is just argument passing (copy reference into parameter slot).

```js
function apply(fn, value) {  // L1 — fn parameter slot receives a function reference
  return fn(value);          // L2 — call via the reference in fn's slot
}                            // L3

const double = (x) => x * 2;  // L4
console.log(apply(double, 5)); // L5 — 10
```

At L5: `double` (a reference) is copied into `fn`'s parameter slot. L2 calls through that reference. No indirection beyond what any object argument has.

### Why this matters

The power isn't in passing one function — it's in **parameterizing behavior**. A HOF separates *what to do* (the passed function) from *when/how to do it* (the HOF's structure). This is the single idea the entire course builds on:

| Pattern | "What" (passed in) | "When/how" (HOF structure) |
|---------|--------------------|-----------------------------|
| `arr.map(fn)` | transform one element | iterate, collect results |
| `arr.filter(fn)` | decide keep/discard | iterate, collect kept |
| `setTimeout(fn, ms)` | the work | schedule after delay |
| `addEventListener(type, fn)` | the handler | dispatch on event |
| `app.get(path, fn)` | request handler | route matching |

Same structural split every time. The HOF owns the *control flow*; the callback owns the *logic*.

### Reference, not call

A common early mistake (not yours at this level, but worth naming for completeness):

```js
setTimeout(greet("A"), 1000);  // BUG: calls greet immediately, passes return value
setTimeout(greet, 1000);       // CORRECT: passes the function reference
```

The distinction: `greet` is a reference (the value in the slot). `greet("A")` is a call expression — it evaluates *now* and produces the return value. Passing to a HOF means passing the reference, not invoking.

---

## Returning functions (factories & closures)

The second HOF shape: a function that *returns* a function. The mechanism is the same — copy a reference into the return slot — but the interesting part is what the returned function **captures**.

```js
function multiplier(factor) {       // L1 — factory
  return (x) => x * factor;         // L2 — returned function closes over factor
}                                    // L3

const triple = multiplier(3);        // L4 — triple holds reference to the arrow from L2
const tenX = multiplier(10);         // L5 — different call → different closure → different factor
console.log(triple(4));              // L6 — 12
console.log(tenX(4));                // L7 — 40
```

### Why this works: closure mechanics (review)

You know this from js-vars-scope, so just the one-line version:

The arrow at L2 is created inside `multiplier`'s execution context. Its `[[Environment]]` internal slot points to `multiplier`'s Function Environment Record — which holds `factor`. When `multiplier` returns, its EC pops off the stack, but the ER stays alive (heap-allocated, GC'd when no closures reference it). The returned arrow can still resolve `factor` via `[[OuterEnv]]` chain walk.

Each call to `multiplier` creates a **new** ER with its own `factor` binding. So `triple` and `tenX` close over *different* ERs — independent state, no sharing.

### The factory pattern

This is the fundamental pattern for **configurable behavior**:

1. Outer function accepts *configuration* (the parameters that vary between uses).
2. Inner function accepts *per-call data* (the arguments that vary on each invocation).
3. Closure bridges the two — configuration is "baked in" without being global.

```js
// Configuration: base URL
// Per-call: endpoint path
function apiClient(baseUrl) {
  return (path) => fetch(`${baseUrl}${path}`);
}

const github = apiClient("https://api.github.com");
const internal = apiClient("https://internal.corp/api");

github("/users/octocat");   // fetches from github
internal("/health");         // fetches from internal
```

This is partial application by hand — you'll see the formal version in chunk 5 (currying & partial application). For now: returning a function lets you split a multi-argument operation into stages, with closure holding the intermediate state.

### Python parallel

Identical mechanism — `def` inside `def`, inner closes over outer's locals. Python's `nonlocal` keyword is needed only for *rebinding* a closed-over variable; reading works without it. JS has no equivalent keyword because JS closures always close over the *binding* (the ER slot), not the value — rebinding just works.





