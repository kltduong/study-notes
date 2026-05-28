# Immutability Patterns — teaching draft

## Plan (teaching order)

- [x] 1. Teaser — mutation bug invisible at the call site
- [x] 2. Pure functions — definition, referential transparency, why purity enables composition
- [ ] 3. Shallow copy idioms — spread, `Object.assign`, `Array.from`, `structuredClone`; shallow vs deep
- [ ] 4. Structural sharing concept — why "copy everything" is O(n) and what persistent data structures do differently
- [ ] 5. When immutability helps vs hurts — decision framework

---

## Teaser — mutation bug invisible at the call site

```js
const addDiscount = (user) => {          // L1
  user.price *= 0.9;                     // L2
  return user;                           // L3
};                                       // L4

const users = [                          // L5
  { name: "A", price: 100 },            // L6
  { name: "B", price: 200 },            // L7
];                                       // L8

const discounted = users.map(addDiscount);  // L9

console.log(discounted[0].price);        // L10
console.log(users[0].price);             // L11
```

### Reveal

Both L10 and L11 print `90`.

`map` creates a new array, but its elements are references to the same objects as the original. `addDiscount` mutates the object through that shared reference (`user.price *= 0.9`), so the mutation is visible through both `discounted[0]` and `users[0]` — they point to the same object in memory.

The call site (`users.map(addDiscount)`) looks safe — `map` "creates a new array," and `addDiscount` returns a value. Nothing at the call site signals that `users` is being destroyed. The bug is invisible without reading `addDiscount`'s implementation.

This is the class of bug that immutability patterns eliminate. The fix: `addDiscount` should return a *new* object instead of mutating the input:

```js
const addDiscount = (user) => ({ ...user, price: user.price * 0.9 });
```

Now `users` is untouched. The function is **pure** — same input always produces the same output, no side effects. The next sub-part formalizes what "pure" means and why it matters for composition.



---

## Pure functions — the axiom that makes composition safe

### Two rules (necessary and sufficient)

A function is **pure** if and only if:

1. **Deterministic.** Same inputs → same output. Always. No dependence on external mutable state (clock, global variable, database, random).
2. **No side effects.** The function doesn't modify anything outside its own scope — no mutation of inputs, no writes to external state, no I/O.

That's it. Two rules. Everything else (referential transparency, testability, composability, memoizability) is a *consequence* of these two.

### Referential transparency — the consequence that matters most

If a function is pure, you can replace any call `f(x)` with its return value without changing program behavior. This property is called **referential transparency**.

```js
// Pure
const double = (x) => x * 2;

const a = double(5);   // → 10
const b = 10;           // → 10 (replaced the call with its return value — program unchanged)
```

You can't do this with impure functions:

```js
let counter = 0;
const increment = () => ++counter;   // impure — depends on + modifies external state

const a = increment();  // → 1 (and counter is now 1)
const b = 1;             // → 1 (but counter is still 0 — program changed)
```

Replacing `increment()` with `1` gives the same *value* for `b`, but the program's state diverges — `counter` never advances. The substitution broke something. With `double`, nothing breaks because the function touches nothing outside itself.

Referential transparency is what makes equational reasoning possible — you can think about functions as values, not as procedures with hidden state.

### Why purity enables composition

`pipe(f, g, h)` assumes each function is a black box: takes input, returns output, nothing else happens. If `f` mutates shared state that `g` reads, the pipeline's behavior depends on *execution order* — reordering, parallelizing, or memoizing any step could break it.

Pure functions compose freely because:

- **Order-independent** (for independent sub-expressions) — no shared mutable state means no ordering dependencies beyond data flow.
- **Memoizable** — same input always gives same output, so you can cache results.
- **Testable** — no setup/teardown of external state; assert on return value alone.
- **Parallelizable** — no data races when there's nothing shared to race on.

The teaser's bug was a composition failure: `map(addDiscount)` looked like a pure pipeline step but secretly destroyed the input. Making `addDiscount` pure (return new object, don't mutate) restores the composition guarantee.

### The purity spectrum in practice

Real programs need side effects — I/O, DOM updates, network calls. The strategy isn't "everything pure" but **push effects to the edges**:

```
Pure core (transforms, business logic, data pipelines)
  ↕
Impure shell (I/O, DOM, network, user input)
```

The pure core is where composition, testing, and reasoning pay off. The impure shell is thin — it reads input, calls the pure core, and writes output. This is the "functional core, imperative shell" architecture.

### Recognizing impurity — the checklist

A function is impure if it does any of:

| Impurity | Example |
|---|---|
| Reads mutable external state | `() => globalConfig.theme` |
| Writes external state | `counter++`, `arr.push(x)` |
| Mutates its arguments | `user.price *= 0.9` |
| Performs I/O | `console.log`, `fetch`, `fs.readFile` |
| Depends on non-deterministic input | `Math.random()`, `Date.now()` |

> **Aside —** `console.log` is technically impure (I/O side effect), but in practice nobody treats debug logging as a purity violation worth refactoring around. The checklist is for reasoning about *data flow correctness*, not for purity dogma.

