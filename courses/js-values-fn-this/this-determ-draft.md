# `this` Determination — Draft

## Plan (teaching order)

- [x] Sub-part 1: The single rule formalized — EvaluateCall reads `[[Base]]`, passes it as thisValue to the new Function EC
- [x] Sub-part 2: Where `[[ThisValue]]` lands — the Function ER slot, accessible via `this` keyword
- [ ] Sub-part 3: Strict vs sloppy coercion — the OrdinaryCallBindThis step
- [ ] Sub-part 4: Complete decision tree — all normal-call cases derived from one rule

---

## Teaser

```js
"use strict";

const obj = {
  name: "Alice",
  greet() { return this.name; },
  nested: {
    name: "Bob",
    greet() { return this.name; }
  }
};

function callWith(fn) {
  return fn();                          // L1
}

console.log(obj.greet());              // L2
console.log(obj.nested.greet());       // L3
console.log(callWith(obj.greet));      // L4

const { greet } = obj;
console.log(greet());                  // L5
```

**Prediction & reveal:** All lines reduce to the same mechanism — what Reference does the call operator see?

- **L2: `obj.greet()`**
  1. Evaluate `obj` — identifier resolution → `Ref₁ { base: scriptER, name: "obj" }`
  2. Call operator needs the object → GetValue(`Ref₁`) → follows scriptER["obj"] → returns the `obj` object. `Ref₁` consumed.
  3. Evaluate `.greet` on that object → `Ref₂ { base: obj, name: "greet" }`
  4. Call operator sees `Ref₂`. `[[Base]]` is `obj` (an object) → `thisValue = obj`
  5. GetValue(`Ref₂`) → `obj["greet"]` → the function object
  6. Call the function with `this = obj` → `this.name` → `"Alice"`

- **L3: `obj.nested.greet()`**
  1. Evaluate `obj` → `Ref₁ { base: scriptER, name: "obj" }`
  2. GetValue(`Ref₁`) → the `obj` object. `Ref₁` consumed.
  3. Evaluate `.nested` → `Ref₂ { base: obj, name: "nested" }`
  4. GetValue(`Ref₂`) → `obj["nested"]` → the nested object. `Ref₂` consumed.
  5. Evaluate `.greet` on nested object → `Ref₃ { base: obj.nested, name: "greet" }`
  6. Call operator sees `Ref₃`. `[[Base]]` is `obj.nested` (an object) → `thisValue = obj.nested`
  7. GetValue(`Ref₃`) → the function object
  8. Call the function with `this = obj.nested` → `this.name` → `"Bob"`

  Key: each dot consumes the previous Reference (GetValue) and produces a new one. Only the **final** Reference reaches the call operator.

- **L4: `callWith(obj.greet)` → inside: `fn()`** — two call sites:

  *Outer call site: `callWith(obj.greet)`*
  1. Evaluate `callWith` → `Ref₁ { base: scriptER, name: "callWith" }`
  2. Call operator sees `Ref₁`. `[[Base]]` is scriptER → `thisValue = undefined`
  3. GetValue(`Ref₁`) → the `callWith` function object
  4. Evaluate the argument `obj.greet`:
     - Evaluate `obj` → `Ref₂ { base: scriptER, name: "obj" }` → GetValue → the obj object. `Ref₂` consumed.
     - Evaluate `.greet` → `Ref₃ { base: obj, name: "greet" }`
     - Argument passing calls GetValue(`Ref₃`) → extracts the greet function object. `Ref₃` consumed.
  5. The greet function object (plain value, no Reference) is copied into `callWith`'s parameter slot `fn`.

  *Inner call site: `fn()` inside `callWith`*
  6. Evaluate `fn` → identifier resolution in callWith's ER → `Ref₄ { base: callWith's ER, name: "fn" }`
  7. Call operator sees `Ref₄`. `[[Base]]` is an ER → `thisValue = undefined`
  8. GetValue(`Ref₄`) → the greet function object
  9. Call the function with `this = undefined` → `this.name` → TypeError

  The function object is the same `greet` throughout — but `Ref₃` (with `base: obj`) was consumed by argument passing. `Ref₄` (with ER base) is a completely new Reference produced by resolving the local identifier `fn`.

- **L5: `const { greet } = obj;` then `greet()`** — destructuring + call:

  *Destructuring step:*
  1. Evaluate `obj` → `Ref₁ { base: scriptER, name: "obj" }` → GetValue → the obj object. `Ref₁` consumed.
  2. Destructuring `{ greet }` reads property `"greet"` from the object → internally evaluates the equivalent of `obj.greet` → `Ref₂ { base: obj, name: "greet" }`
  3. Assignment into the local binding calls GetValue(`Ref₂`) → extracts the greet function object. `Ref₂` consumed.
  4. Stores the function object in scriptER slot `greet`.

  *Call step: `greet()`*
  5. Evaluate `greet` → identifier resolution → `Ref₃ { base: scriptER, name: "greet" }`
  6. Call operator sees `Ref₃`. `[[Base]]` is scriptER → `thisValue = undefined`
  7. GetValue(`Ref₃`) → the greet function object
  8. Call the function with `this = undefined` → `this.name` → TypeError

  Same mechanism as L4: the operation that stores the function (destructuring assignment) calls GetValue, consuming the Reference that had `obj` as base. The subsequent call resolves `greet` fresh — gets an ER base — `this` is `undefined`.

**Common pattern across L4 and L5:** Any operation that extracts the function from its member-expression context (argument passing, assignment, destructuring, `return`) calls GetValue and consumes the Reference. The next call site produces a *new* Reference by resolving whatever identifier now holds the function — and that identifier lives in an ER, not on an object.

---

## Sub-part 1: The single rule formalized

You already know the Reference Record from the previous chunk — `{ [[Base]], [[ReferencedName]], [[Strict]] }`. Now we trace what happens *after* the call operator reads it.

### EvaluateCall — the full sequence

When the engine encounters a call expression — any `expr()` where `expr` is the expression left of the parentheses (an identifier, member expression, comma expression, another call, etc.):

```
1. ref  = evaluate(expr)           — may be a Reference or a plain value
2. func = GetValue(ref)            — always extracts the function object
3. thisValue = ?                   — determined NOW, from ref (before GetValue consumed it)
4. Call func, passing thisValue    — enters the function
```

Step 3 is the entire `this` mechanism for normal calls. The rule:

```
If ref is a Reference Record:
    If [[Base]] is an object       → thisValue = [[Base]]
    If [[Base]] is an ER           → thisValue = undefined
If ref is NOT a Reference Record   → thisValue = undefined
```

That's it. Three branches, one decision point: **what did the call operator see as `ref`?**

### Why ER base → `undefined` (not the ER itself)

An Environment Record is an internal spec construct — not a JS value. You can't pass it as `this`. The spec explicitly says: if the base is an ER, `thisValue` is `undefined`. This is the mechanism behind "plain calls don't have a `this`."

```js
"use strict";
function standalone() { return this; }

standalone();  // undefined
// ref = Reference { base: globalER, name: "standalone", strict: true }
// [[Base]] is an ER → thisValue = undefined
```

The function was found via identifier resolution — the ER where `standalone` lives becomes `[[Base]]`. ER base → `undefined`. No magic, no special "default binding rule" — just the Reference base being an ER.



---

## Sub-part 2: Where `[[ThisValue]]` lands

Sub-part 1 answered *how* `thisValue` is determined (Reference base rule). This sub-part answers: where does that value go, and how does the `this` keyword read it?

### Terminology: `thisValue` vs `[[ThisValue]]` vs `this`

Three names, three layers — all resolve to the same underlying value for a given call, but exist at different moments:

| Term | What it is | Layer |
|------|-----------|-------|
| `thisValue` | Temporary value computed by EvaluateCall — the result of reading `[[Base]]` from the Reference. A local variable in the spec algorithm, not stored anywhere yet. | Call-site evaluation (transient) |
| `[[ThisValue]]` | Internal slot on the Function Environment Record. `thisValue` is written here when the new EC is created. Persists for the lifetime of that call's ER. | Storage (per-call ER slot) |
| `this` | Source-code keyword. Evaluating it runs `ResolveThisBinding()`, which reads `[[ThisValue]]` from the current Function ER. The read interface. | Source-level access (read-only) |

The pipeline: call site produces `thisValue` → stored into `ER.[[ThisValue]]` → `this` keyword reads it back. The draft uses them precisely in this sense throughout.

### The path: call site → Function ER slot → `this` keyword

You know from js-vars-scope that every function call creates a new Execution Context with a new Function Environment Record. That ER has a `[[ThisValue]]` internal slot — and that's exactly where the determined `thisValue` is stored.

```mermaid
flowchart LR
    A["Call operator<br/>determines thisValue<br/>(Reference base rule)"] --> B["New Function EC<br/>created for this call"]
    B --> C["Function ER<br/>[[ThisValue]] ← thisValue"]
    C --> D["this keyword<br/>reads [[ThisValue]]"]

    style A fill:#46c,stroke:#fff,color:#fff
    style B fill:#46c,stroke:#fff,color:#fff
    style C fill:#2a2,stroke:#fff,color:#fff
    style D fill:#2a2,stroke:#fff,color:#fff
```

The `this` keyword doesn't do a lookup or chain walk. It's a direct slot read:

```
ResolveThisBinding():
    env = current Function ER
    return env.[[ThisValue]]
```

### `this` is per-call, not per-function

Because `[[ThisValue]]` lives in the ER (created fresh per call), the same function object gets different `this` values across different calls:

```js
"use strict";
function show() { return this; }

const a = { show };
const b = { show };

a.show();  // Ref { base: a } → this = a     → new ER1, [[ThisValue]] = a
b.show();  // Ref { base: b } → this = b     → new ER2, [[ThisValue]] = b
show();    // Ref { base: scriptER } → this = undefined → new ER3, [[ThisValue]] = undefined
```

Three calls, three ECs, three ER slots, three different `this` values. The function object `show` is identical in all three — it doesn't carry `this`. The call site determines it; the ER stores it.

### Why this matters: `this` is not a property of the function

This is the structural reason "method extraction loses `this`." The function object has no `[[ThisValue]]` — that slot belongs to the *ER created at call time*. When you copy a function reference into a different binding and call it from there, a new ER is created with a new `[[ThisValue]]` determined by the *new* call site's Reference base. The old call site's `this` is gone — it lived in a different ER that no longer exists.

```js
"use strict";
const obj = { name: "obj", show() { return this; } };

const fn = obj.show;  // copy function reference into fn's slot
fn();                 // new call → new ER → [[ThisValue]] = undefined (ER base)
obj.show();           // new call → new ER → [[ThisValue]] = obj (object base)
```

Each `()` creates its own ER with its own `[[ThisValue]]`. There's no "memory" of previous calls.
