# Explicit Overrides: `call`, `apply`, `bind` — Draft

## Plan (teaching order)

- [x] `call` and `apply` — mechanism: bypass Reference-base rule by supplying `thisValue` directly
- [x] `bind` — mechanism: BoundFunction exotic object, `[[BoundThis]]` + `[[BoundArguments]]`
- [ ] Priority / interaction — why `bind` beats `call`/`apply` (wrapper intercepts)
- [ ] Partial application — `bind` with pre-filled arguments
- [ ] Edge cases and gotchas

---

## Part 1: `call` and `apply` — supplying `this` directly

### Where they fit in the pipeline

Recall the `this`-determination pipeline from the previous chunk:

```
call site → thisValue (from Reference base) → OrdinaryCallBindThis → ER.[[ThisValue]]
```

`call` and `apply` **replace step 1**. Instead of the engine reading `[[Base]]` from a Reference, *you* supply `thisValue` explicitly. The rest of the pipeline (OrdinaryCallBindThis coercion, ER slot storage) runs identically.

### The spec mechanism

`Function.prototype.call(thisArg, ...args)` does:

```
1. Let func = this (the function .call was invoked on)
2. Call func.[[Call]](thisArg, args)
```

That's it. It invokes the function's internal `[[Call]]` method directly — the same internal method the `()` operator uses, but with a `thisValue` you chose instead of one derived from a Reference.

`apply` is identical except arguments come as a single array-like:

```js
func.call(thisArg, arg1, arg2)    // spread args
func.apply(thisArg, [arg1, arg2]) // array-like args
```

The `this`-binding mechanism is the same. The only difference is argument delivery shape.

### Why this works

The call operator's job is:

1. Determine `thisValue` (from Reference base)
2. Invoke `[[Call]](thisValue, args)` on the function

`call`/`apply` skip step 1 and go straight to step 2 with your supplied value. They don't "override" anything — they just enter the pipeline at a different point.

```js
"use strict";
function greet(greeting) { return `${greeting}, ${this.name}`; }

const obj = { name: "obj" };

// Normal call — Reference base rule:
obj.greet = greet;
obj.greet("Hi");           // Ref { base: obj } → this = obj → "Hi, obj"

// call — you supply thisValue directly:
greet.call(obj, "Hi");     // thisValue = obj → "Hi, obj"

// Same result, different entry point into the pipeline.
```

### OrdinaryCallBindThis still runs

`call`/`apply` don't skip the coercion step:

```js
// sloppy mode
function sloppy() { return this; }
sloppy.call(undefined);  // → globalThis (coerced)
sloppy.call(null);       // → globalThis (coerced)
sloppy.call(42);         // → Number {42} (wrapped)

// strict mode
function strict() { "use strict"; return this; }
strict.call(undefined);  // → undefined (pass-through)
strict.call(null);       // → null (pass-through)
strict.call(42);         // → 42 (pass-through, no wrapping)
```

Same coercion rules as normal calls — `call`/`apply` only replace the *source* of `thisValue`, not the downstream processing.

> **Aside —** `apply`'s main historical use case (spreading an array into arguments) is largely obsolete since ES6 spread syntax. You'll still see it in older code and in one niche: forwarding an `arguments` object without knowing its length.

---

## Part 2: `bind` — the BoundFunction wrapper

### Teaser

```js
"use strict";
function greet() { return this.name; }

const bound = greet.bind({ name: "A" });

bound.call({ name: "B" });  // → "A"
```

`call` faithfully delivers `{ name: "B" }` — but it delivers it to the *wrapper*, not to `greet`. The wrapper intercepts and substitutes.

### What `bind` actually returns

`Function.prototype.bind(thisArg, ...args)` does **not** mutate or annotate the original function. It creates and returns a new object — a **BoundFunction exotic object**. "Exotic" means it has a custom `[[Call]]` internal method (unlike ordinary functions which all share the same `[[Call]]` algorithm).

The BoundFunction stores three internal slots:

| Slot | Value | Purpose |
|------|-------|---------|
| `[[BoundTargetFunction]]` | The original function (`greet`) | What to actually invoke |
| `[[BoundThis]]` | The `thisArg` you passed to `bind` | Replaces any incoming `thisValue` |
| `[[BoundArguments]]` | Any extra args passed to `bind` | Prepended to call-time arguments (partial application) |

### The BoundFunction `[[Call]]` algorithm

When anything invokes a BoundFunction (normal call, `call`, `apply`, another `bind`, doesn't matter):

```
BoundFunction.[[Call]](thisArgument, argumentsList):
    1. Let target = [[BoundTargetFunction]]
    2. Let boundThis = [[BoundThis]]           ← ignores thisArgument entirely
    3. Let boundArgs = [[BoundArguments]]
    4. Let args = concat(boundArgs, argumentsList)
    5. Return target.[[Call]](boundThis, args)  ← calls the real function
```

Step 2 is the key: `thisArgument` (whatever the caller supplied) is **never read**. The wrapper unconditionally uses `[[BoundThis]]`. This isn't a priority system — it's structural interception. The incoming `this` is discarded before it can reach the target.

### Why `call`/`apply` can't override `bind`

```js
"use strict";
function greet() { return this.name; }
const bound = greet.bind({ name: "A" });

bound.call({ name: "B" });
```

Trace:

```
1. .call invokes bound.[[Call]]({ name: "B" }, [])
2. bound is a BoundFunction → its [[Call]] runs:
   - Ignores { name: "B" }
   - Uses [[BoundThis]] = { name: "A" }
   - Calls greet.[[Call]]({ name: "A" }, [])
3. greet runs with this = { name: "A" }
4. → "A"
```

`call` did its job correctly — it passed `{ name: "B" }` as `thisArgument` to the function it was called on. But the function it was called on is the *wrapper*, and the wrapper's `[[Call]]` doesn't forward that argument.

### Double-bind: same principle

```js
const bound1 = greet.bind({ name: "first" });
const bound2 = bound1.bind({ name: "second" });

bound2();  // → "first"
```

`bound2` is a BoundFunction wrapping `bound1`. When called:
- `bound2.[[Call]]` → uses its `[[BoundThis]]` = `{ name: "second" }`, calls `bound1.[[Call]]({ name: "second" }, [])`
- `bound1.[[Call]]` → ignores `{ name: "second" }`, uses its `[[BoundThis]]` = `{ name: "first" }`, calls `greet.[[Call]]({ name: "first" }, [])`
- `greet` sees `this = { name: "first" }`

The innermost `bind` (closest to the original function) always wins — each wrapper discards whatever the outer wrapper passed.

### `bind` doesn't affect the original

```js
const original = greet;
const bound = greet.bind({ name: "bound" });

original();  // TypeError (this = undefined, strict mode)
bound();     // "bound"

original === bound;  // false — different objects entirely
```

`bind` is pure — it returns a new object, leaves the original untouched. No mutation, no shared state.

### The full `this`-determination pipeline (with overrides)

```mermaid
flowchart TD
    A["Expression before ()"] --> Q0{"How is the function invoked?"}
    
    Q0 -->|"Normal call: expr()"| Q1{"Is expr a Reference?"}
    Q0 -->|"func.call(thisArg) /<br/>func.apply(thisArg)"| CA["thisValue = thisArg<br/>(you supply it)"]
    
    Q1 -->|"Yes, object base"| OBJ["thisValue = [[Base]]"]
    Q1 -->|"Yes, ER base / No"| UN["thisValue = undefined"]
    
    OBJ --> BF{"Is the function a<br/>BoundFunction?"}
    UN --> BF
    CA --> BF
    
    BF -->|"Yes"| BIND["Discard thisValue<br/>Use [[BoundThis]] instead"]
    BF -->|"No"| COERCE["OrdinaryCallBindThis<br/>(coerce if sloppy)"]
    
    BIND --> COERCE
    COERCE --> ER["ER.[[ThisValue]] = final value"]
    ER --> KW["this keyword reads slot"]

    style A fill:#46c,stroke:#fff,color:#fff
    style Q0 fill:#555,stroke:#fff,color:#fff
    style Q1 fill:#555,stroke:#fff,color:#fff
    style CA fill:#c42,stroke:#fff,color:#fff
    style OBJ fill:#2a2,stroke:#fff,color:#fff
    style UN fill:#c42,stroke:#fff,color:#fff
    style BF fill:#555,stroke:#fff,color:#fff
    style BIND fill:#a3c,stroke:#fff,color:#fff
    style COERCE fill:#c42,stroke:#fff,color:#fff
    style ER fill:#2a2,stroke:#fff,color:#fff
    style KW fill:#2a2,stroke:#fff,color:#fff
```

**† Legend:**
- Blue: starting point
- Green: `this` successfully bound from object base / stored in ER
- Red: `undefined` path or coercion step
- Purple: `bind` interception — unconditionally replaces whatever came before
- Grey: decision nodes

**Abbreviations:** ER = Environment Record, BF = BoundFunction

`bind` sits *after* both the Reference-base rule and `call`/`apply`. No matter which path produces `thisValue`, if the function is a BoundFunction, that value gets discarded and replaced with `[[BoundThis]]`.
