# Explicit Overrides: `call`, `apply`, `bind` — Draft

## Plan (teaching order)

- [x] `call` and `apply` — mechanism: bypass Reference-base rule by supplying `thisValue` directly
- [ ] `bind` — mechanism: BoundFunction exotic object, `[[BoundThis]]` + `[[BoundArguments]]`
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
