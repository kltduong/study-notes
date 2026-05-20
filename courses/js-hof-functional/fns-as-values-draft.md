# Functions as Values — Draft

## Plan (teaching order)

- [x] First-class axiom — what "first-class" means structurally
- [x] Passing functions — higher-order function definition, callback pattern
- [ ] Returning functions — factory pattern, closures as the enabling mechanism
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

