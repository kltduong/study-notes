# 1. Functions as Values — Draft

## 1.1. Plan (teaching order)

- [x] First-class axiom — what "first-class" means structurally
- [x] Passing functions — higher-order function definition, callback pattern
- [x] Returning functions — factory pattern, closures as the enabling mechanism
- [x] Function identity & reference semantics — equality, aliasing, mutation implications
- [x] Bridge to the course — what this enables (the rest of the course derives from here)

---

## 1.2. First-class axiom

**TL;DR:** "First-class" is not a feature — it's a classification. A value is first-class if it has the same rights as every other value in the language. In JS, functions are objects, so they get object rights automatically.

### 1.2.1. The axiom set

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

### 1.2.2. Python comparison

Python is identical here — functions are first-class objects, `def` creates a callable object, assignment copies a reference. The parallel is exact. The difference shows up later: Python's `lambda` is expression-limited (single expression, no statements), while JS arrow functions are full-bodied. This matters for HOF ergonomics — JS can inline complex logic where Python often needs a named `def`.

### 1.2.3. Teaser reveal

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

## 1.3. Passing functions (higher-order functions & callbacks)

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

### 1.3.1. Why this matters

The power isn't in passing one function — it's in **parameterizing behavior**. A HOF separates *what to do* (the passed function) from *when/how to do it* (the HOF's structure). This is the single idea the entire course builds on:

| Pattern | "What" (passed in) | "When/how" (HOF structure) |
|---------|--------------------|-----------------------------|
| `arr.map(fn)` | transform one element | iterate, collect results |
| `arr.filter(fn)` | decide keep/discard | iterate, collect kept |
| `setTimeout(fn, ms)` | the work | schedule after delay |
| `addEventListener(type, fn)` | the handler | dispatch on event |
| `app.get(path, fn)` | request handler | route matching |

Same structural split every time. The HOF owns the *control flow*; the callback owns the *logic*.

### 1.3.2. Reference, not call

A common early mistake (not yours at this level, but worth naming for completeness):

```js
setTimeout(greet("A"), 1000);  // BUG: calls greet immediately, passes return value
setTimeout(greet, 1000);       // CORRECT: passes the function reference
```

The distinction: `greet` is a reference (the value in the slot). `greet("A")` is a call expression — it evaluates *now* and produces the return value. Passing to a HOF means passing the reference, not invoking.

---

## 1.4. Returning functions (factories & closures)

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

### 1.4.1. Why this works: closure mechanics (review)

You know this from js-vars-scope, so just the one-line version:

The arrow at L2 is created inside `multiplier`'s execution context. Its `[[Environment]]` internal slot points to `multiplier`'s Function Environment Record — which holds `factor`. When `multiplier` returns, its EC pops off the stack, but the ER stays alive (heap-allocated, GC'd when no closures reference it). The returned arrow can still resolve `factor` via `[[OuterEnv]]` chain walk.

Each call to `multiplier` creates a **new** ER with its own `factor` binding. So `triple` and `tenX` close over *different* ERs — independent state, no sharing.

### 1.4.2. The factory pattern

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

### 1.4.3. Python parallel

Identical mechanism — `def` inside `def`, inner closes over outer's locals. Python's `nonlocal` keyword is needed only for *rebinding* a closed-over variable; reading works without it. JS has no equivalent keyword because JS closures always close over the *binding* (the ER slot), not the value — rebinding just works.

---

## 1.5. Function identity & reference semantics

Since functions are objects, all object identity rules apply. This has practical consequences for HOF-heavy code.

### 1.5.1. Identity (`===`)

```js
const f = (x) => x;           // L1
const g = f;                   // L2 — copies reference; same object
const h = (x) => x;           // L3 — new object, same behavior

console.log(f === g);          // L4 — true (same reference)
console.log(f === h);          // L5 — false (different objects)
```

**Consequence:** You cannot compare functions by behavior in JS. Two functions that do the same thing are still `!==` if they're different objects. This matters for:

- **React's dependency arrays** — `useEffect([fn])` re-fires if `fn` is a new object each render. Hence `useCallback`.
- **Event listener removal** — `removeEventListener` matches by reference. Pass a different object → no removal.
- **Set/Map keying** — `new Set([f, f, h])` has size 2, not 1 by behavior.

### 1.5.2. Aliasing

```js
function original() { return 1; }  // L1
const alias = original;             // L2 — alias and original: same object

alias.custom = "tagged";            // L3 — mutates the shared object
console.log(original.custom);       // L4 — "tagged" (same object)
```

Functions are mutable objects — you can add properties. Aliasing means mutations are visible through all references. This is rarely useful intentionally, but it's the mechanism behind things like `express` middleware attaching properties to handler functions, or test frameworks tagging functions with metadata.

### 1.5.3. Functions created in loops / HOFs

```js
const fns = [];
for (let i = 0; i < 3; i++) {
  fns.push(() => i);             // L1 — each iteration: new arrow, new closure
}
console.log(fns[0] === fns[1]);  // L2 — false (different objects)
console.log(fns[0](), fns[1](), fns[2]()); // L3 — 0, 1, 2 (each closes over its own `i`)
```

Each iteration of the loop evaluates the arrow expression → new function object → new closure over that iteration's block-scoped `i`. Three distinct objects, three distinct closures. (With `var` instead of `let`, you'd get three distinct objects that all close over the *same* binding — the classic bug. But that's a scope issue, not a first-class-function issue.)


---

## 1.6. Bridge: what this enables

Everything in this course is a consequence of the two axioms:

1. **Functions are objects** (callable, heap-allocated, have identity).
2. **One value-passing rule** (slot content is copied; for objects, that's a reference).

From these, the rest of the course unfolds:

| Course chunk | Derives from |
|---|---|
| **map/filter/reduce** | Pass a function → parameterize iteration behavior |
| **Composition** | Return a function that calls two others in sequence |
| **Currying** | Return a function that closes over the first argument |
| **Memoization** | Return a wrapper function that closes over a cache |
| **Decorators** | Accept a function, return a new function that wraps it |

No new language mechanism is introduced. Every pattern in this course is a *usage pattern* built on first-class functions + closures. The complexity is in the *design* (which function to pass, what to close over, how to compose), not in the *mechanism*.

> 🔖 **Later:** The one place where a new mechanism *does* appear is generators (`function*`) — they're callable objects with suspend/resume semantics. That's a different course (js-iterators-generators). Everything here uses plain functions.
