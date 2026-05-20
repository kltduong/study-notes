# 1. Functions as Values

> **TL;DR:** "First-class" means functions are objects — callable, heap-allocated, identity-bearing. Combined with JS's single value-passing rule (copy the slot; for objects that's a reference), every pattern in this course follows: HOFs, closures-as-factories, composition, currying. No new mechanism — just usage patterns on top of two axioms.

## 1.1. The two axioms

1. **Functions are objects.** Callable objects with a `[[Call]]` internal slot. Heap-allocated, have properties, a prototype chain, identity.
2. **One value-passing rule.** Assignment, argument passing, return — all copy the slot content. For objects (including functions), the slot holds a reference.

Everything below derives from these two facts.

## 1.2. Derived capabilities

| Operation | Why it works |
|-----------|-------------|
| Assign to variable / store in array | Slot holds reference — same as any object |
| Pass as argument | Copies reference into parameter slot |
| Return from function | Copies reference into caller's slot |
| Compare with `===` | Compares references (identity, not behavior) |
| Add properties | Object → has property map |

No special "function-passing" mechanism exists. It's just object rules.

## 1.3. Higher-order functions

A HOF takes a function as argument, returns a function, or both. The structural insight: a HOF separates **what** (the passed function — the logic) from **when/how** (the HOF's own control flow).

| Pattern | "What" (passed in) | "When/how" (HOF structure) |
|---------|--------------------|-----------------------------|
| `arr.map(fn)` | transform one element | iterate, collect results |
| `arr.filter(fn)` | decide keep/discard | iterate, collect kept |
| `arr.sort(cmp)` | ordering logic per pair | sorting algorithm |
| `setTimeout(fn, ms)` | the work | schedule after delay |
| `addEventListener(type, fn)` | the handler | dispatch on event |

## 1.4. Returning functions: the factory pattern

A function that returns a function splits a multi-argument operation into stages:

1. **Outer** accepts *configuration* (varies between uses).
2. **Inner** accepts *per-call data* (varies on each invocation).
3. **Closure** bridges the two — configuration is baked in via the outer call's ER.

```js
function multiplier(factor) {
  return (x) => x * factor;   // closes over factor in multiplier's ER
}
const triple = multiplier(3);  // triple's [[Environment]] → ER with factor=3
const tenX = multiplier(10);   // different call → different ER → factor=10
```

Each call to the factory creates a new Function ER. Returned closures from different calls are independent — they close over different ERs.

> **Aside — Python parallel.** Identical mechanism: `def` inside `def`, inner closes over outer's locals. The difference: Python needs `nonlocal` to *rebind* a closed-over variable; JS closures always close over the binding (the ER slot), so rebinding just works.

## 1.5. Function identity

`===` compares references, not behavior. Two functions with identical bodies are `!==` if they're different objects.

```js
const f = (x) => x;
const g = f;          // same object
const h = (x) => x;  // different object, same behavior

f === g  // true
f === h  // false
```

Practical consequences:
- **`removeEventListener`** matches by reference — must pass the same object used in `addEventListener`.
- **React dependency arrays** — new function object each render triggers re-runs. Hence `useCallback`.
- **Set/Map keying** — identity-based, not behavior-based.

### 1.5.1. Aliasing

Assigning a function to another variable copies the reference. Both names point to the same mutable object — property mutations are visible through either.

### 1.5.2. Functions in loops

Each evaluation of a function expression creates a new object. In a `let`-scoped loop, each iteration's arrow closes over its own block-scoped binding — independent closures, independent state.

## 1.6. Bridge: what this enables

Every pattern in this course is a usage pattern built on these two axioms + closures:

| Course chunk | Derives from |
|---|---|
| map/filter/reduce | Pass a function → parameterize iteration |
| Composition | Return a function that calls two others in sequence |
| Currying | Return a function that closes over the first argument |
| Memoization | Return a wrapper that closes over a cache |
| Decorators | Accept a function, return a wrapping function |

No new language mechanism is introduced. The complexity is in the *design* (which function to pass, what to close over, how to compose), not in the mechanism.

> 🔖 **Later:** Generators (`function*`) add suspend/resume semantics — a genuinely new mechanism. That's js-iterators-generators, not this course.

## 1.7. Quick reference

- **First-class** — functions are objects; they follow the same reference/identity/passing rules as any object.
- **HOF structural split** — HOF owns control flow ("when/how"); callback owns logic ("what").
- **Factory pattern** — outer accepts config, returns inner that accepts per-call data; closure bridges the two via the outer's ER.
- **Identity, not behavior** — `===` compares references; two identical-body functions are `!==` if different objects. Matters for event listeners, React deps, Set/Map keys.
- **Two axioms** — (1) functions are objects, (2) one value-passing rule. The entire course derives from these.
