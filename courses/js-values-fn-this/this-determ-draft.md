# `this` Determination — Draft

## Plan (teaching order)

- [x] Sub-part 1: The single rule formalized — EvaluateCall reads `[[Base]]`, passes it as thisValue to the new Function EC
- [x] Sub-part 2: Where `[[ThisValue]]` lands — the Function ER slot, accessible via `this` keyword
- [x] Sub-part 3: Strict vs sloppy coercion — the OrdinaryCallBindThis step
- [x] Sub-part 4: Complete decision tree — all normal-call cases derived from one rule

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

---

## Sub-part 3: Strict vs sloppy coercion — OrdinaryCallBindThis

Sub-parts 1–2 gave the full pipeline: call site → `thisValue` → `ER.[[ThisValue]]` → `this` keyword. But there's one more step between "call site computes `thisValue`" and "ER stores it" — a coercion step that only fires in sloppy mode.

### Teaser

```js
function sloppy() { return this; }

function strict() { "use strict"; return this; }

console.log(sloppy());  // L1
console.log(strict());  // L2
```

Both are plain calls. Identifier resolution → ER base → `thisValue = undefined`. Same rule, same result. Yet L1 returns `globalThis` and L2 returns `undefined`.

### The missing step: OrdinaryCallBindThis

The pipeline from sub-part 2 was slightly simplified. The full sequence when a function is called:

```
1. Call site: thisValue = Reference base rule        (sub-part 1)
2. New EC created
3. OrdinaryCallBindThis(func, thisValue)             ← THIS STEP
4. ER.[[ThisValue]] = result of step 3               (sub-part 2)
5. this keyword reads ER.[[ThisValue]]               (sub-part 2)
```

Step 3 is where strict/sloppy diverges. The algorithm:

```
OrdinaryCallBindThis(F, thisValue):
    if F.[[ThisMode]] is "strict":
        actualThis = thisValue                    // pass through unchanged
    else:  // sloppy
        if thisValue is undefined or null:
            actualThis = globalThis               // coerce to global object
        else:
            actualThis = ToObject(thisValue)      // wrap primitives
    Store actualThis in the new ER's [[ThisValue]]
```

### What `[[ThisMode]]` is

Every function object has a `[[ThisMode]]` internal slot, set at function creation time:

| Function type | `[[ThisMode]]` | Set when |
|---|---|---|
| Strict-mode function | `"strict"` | Function is created inside strict code |
| Sloppy-mode function | `"global"` | Function is created in non-strict code |
| Arrow function | `"lexical"` | Always (arrows skip OrdinaryCallBindThis entirely) |

This is a property of the *function object* — not the call site, not the Reference. It's baked in at creation time and never changes.

### The coercion rules in sloppy mode

When `[[ThisMode]]` is `"global"` (sloppy), OrdinaryCallBindThis does two things:

1. **`undefined` / `null` → `globalThis`** — this is why `sloppy()` returns the global object instead of `undefined`. The engine "helpfully" replaces the missing `this` with the global object.

2. **Primitive → wrapped object** — if `thisValue` is a primitive (number, string, boolean, symbol, bigint), it gets wrapped via `ToObject`:

```js
function sloppy() { return this; }

sloppy.call(42);       // Number {42} — wrapped
sloppy.call("hi");     // String {"hi"} — wrapped
sloppy.call(true);     // Boolean {true} — wrapped
```

In strict mode, neither coercion happens — `undefined` stays `undefined`, primitives stay primitives:

```js
function strict() { "use strict"; return this; }

strict.call(42);       // 42 — raw primitive
strict.call(null);     // null — no coercion
strict.call(undefined);// undefined — no coercion
```

### Why this exists (historical accident)

This is genuinely arbitrary — a legacy design choice from ES1 (1997), not derivable from any principle. The original reasoning: "methods should always have an object as `this`" — so the engine auto-wraps. But this creates bugs:

```js
function addProp() { this.tagged = true; }

addProp();  // sloppy: this = globalThis → silently pollutes the global object
            // strict: this = undefined → TypeError (the correct failure)
```

Strict mode (ES5) fixed this by removing the coercion. The function gets exactly what the call site determined — no "helpful" patching.

> **Aside —** This is one of the strongest arguments for always using strict mode (or modules/classes, which are strict by default). The sloppy coercion masks bugs by silently substituting `globalThis` where `undefined` would correctly throw.

### The complete pipeline (revised)

```mermaid
flowchart LR
    A["Call site<br/>thisValue = Ref base rule"] --> B["OrdinaryCallBindThis<br/>coerce if sloppy"]
    B --> C["ER.[[ThisValue]]<br/>= final value"]
    C --> D["this keyword<br/>reads slot"]

    style A fill:#46c,stroke:#fff,color:#fff
    style B fill:#c42,stroke:#fff,color:#fff
    style C fill:#2a2,stroke:#fff,color:#fff
    style D fill:#2a2,stroke:#fff,color:#fff
```

**† Legend:**
- Blue: determination (sub-part 1)
- Red: coercion step (sub-part 3) — only mutates the value in sloppy mode
- Green: storage and read (sub-part 2)

### Summary: what strict mode changes (and doesn't)

| Aspect | Strict | Sloppy |
|--------|--------|--------|
| Reference base rule | Same | Same |
| ER base → `thisValue = undefined` | Same | Same |
| OrdinaryCallBindThis coercion | **No** — pass through | **Yes** — `undefined`/`null` → `globalThis`, primitives → wrapped |
| Where the check lives | `F.[[ThisMode]]` on the function object | Same |

The call-site rule is identical. The only difference is whether the result gets patched before storage. Strict mode is the "honest" path — what the call site determined is what you get.

---

## Sub-part 4: Complete decision tree — all normal-call cases derived from one rule

Sub-parts 1–3 gave you the mechanism. This sub-part is the payoff: a single decision tree that covers every normal-call pattern you'll encounter. No new concepts — just the one rule applied systematically.

### Teaser

```js
"use strict";

const obj = {
  x: 1,
  getX() { return this.x; }
};

const cases = [
  () => obj.getX(),                    // A
  () => (0, obj.getX)(),               // B
  () => { const f = obj.getX; return f(); },  // C
  () => (obj.getX)(),                  // D
];

cases.forEach((fn, i) => {
  try { console.log(String.fromCharCode(65+i) + ":", fn()); }
  catch(e) { console.log(String.fromCharCode(65+i) + ": TypeError"); }
});
```

Results: A → `1`, B → TypeError, C → TypeError, D → `1`.

- **A:** `obj.getX` → `Ref { base: obj }` → call operator sees object base → `this = obj`
- **B:** Comma operator evaluates `obj.getX` → `Ref { base: obj }` → calls GetValue → plain function value. Call operator sees a non-Reference → `this = undefined`
- **C:** Assignment `const f = obj.getX` calls GetValue on the Reference → stores plain function in `f`. `f()` → identifier resolution → `Ref { base: ER }` → `this = undefined`
- **D:** Grouping `(obj.getX)` does **not** call GetValue — spec explicitly says it returns the operand as-is. Call operator still sees `Ref { base: obj }` → `this = obj`

### The decision tree

Every normal call (no `call`/`apply`/`bind`, no `new`, no arrow) reduces to one question:

```
What does the call operator see as `ref`?
│
├─ ref is a Reference Record
│   ├─ [[Base]] is an object  → thisValue = that object
│   └─ [[Base]] is an ER      → thisValue = undefined
│
└─ ref is NOT a Reference     → thisValue = undefined
    (plain value — GetValue was already called by something)
```

Then OrdinaryCallBindThis applies (strict: pass-through; sloppy: coerce `undefined`→`globalThis`).

### What produces each case

| `ref` state | How you get there | Example |
|---|---|---|
| Reference with object base | Member expression directly before `()` | `obj.method()`, `arr[0]()`, `obj["m"]()` |
| Reference with ER base | Identifier directly before `()` | `fn()`, `myFunc()`, `imported()` |
| Not a Reference (plain value) | Any operation that calls GetValue before `()` reaches the result | `(0, obj.m)()`, `(f = obj.m)()`, `fn()` after `const fn = obj.m` |

### Operations that consume (GetValue) a Reference

These are the "Reference killers" — they extract the value and discard the Reference wrapper:

- **Assignment** (`=`, `+=`, destructuring) — RHS is GetValue'd before storing
- **Argument passing** — each argument expression is GetValue'd before the value enters the parameter slot
- **Comma operator** — evaluates and GetValue's each operand
- **Conditional operator** (`? :`) — GetValue's the selected branch
- **Logical operators** (`&&`, `||`, `??`) — GetValue the result when short-circuiting resolves
- **`return`** — GetValue's the expression

### Operations that do NOT consume a Reference

- **Grouping `( )`** — transparent, returns operand as-is
- **Member access** (`.` or `[]`) — consumes the *previous* Reference (to get the object), but *produces a new one* with that object as base

### The "method extraction" pattern — unified explanation

Every case where "a method loses `this`" is the same mechanism: something called GetValue on the member-expression Reference before the call operator saw it.

```js
"use strict";
const obj = { name: "obj", greet() { return this.name; } };

// Direct call — Reference reaches call operator intact
obj.greet();                    // "obj"

// Extraction patterns — GetValue fires before ()
const fn = obj.greet; fn();     // undefined (assignment consumed Ref)
[obj.greet][0]();               // this = the array (new Ref with array base)
(true && obj.greet)();          // undefined (&& GetValue'd the result)
(obj.greet || null)();          // undefined (|| GetValue'd the result)
setTimeout(obj.greet, 0);       // undefined (argument passing consumed Ref)
```

All one rule. No special cases to memorize.

### Edge case: computed member expression with side effects

```js
"use strict";
let count = 0;
const obj = {
  get prop() { count++; return function() { return this; }; }
};

obj.prop();  // this = obj
```

Even though `prop` is a getter that runs code, the *member expression* `obj.prop` still produces `Ref { base: obj, name: "prop" }`. The call operator reads `[[Base]]` = `obj` → `thisValue = obj`. GetValue is called to get the function (which triggers the getter), but the Reference's base was already read for `this` determination. The getter's side effects don't affect `this`.

### Decision tree as a flowchart

```mermaid
flowchart TD
    Start["Expression before ()"] --> Q1{"Is it a<br/>Reference Record?"}
    Q1 -->|"No (plain value)"| U["thisValue = undefined"]
    Q1 -->|"Yes"| Q2{"[[Base]] type?"}
    Q2 -->|"Object"| OBJ["thisValue = [[Base]]"]
    Q2 -->|"Environment Record"| U
    U --> Coerce{"Function strict?"}
    OBJ --> Store["ER.[[ThisValue]] = thisValue"]
    Coerce -->|"Strict"| Store2["ER.[[ThisValue]] = undefined"]
    Coerce -->|"Sloppy"| Store3["ER.[[ThisValue]] = globalThis"]

    style Start fill:#46c,stroke:#fff,color:#fff
    style Q1 fill:#555,stroke:#fff,color:#fff
    style Q2 fill:#555,stroke:#fff,color:#fff
    style U fill:#c42,stroke:#fff,color:#fff
    style OBJ fill:#2a2,stroke:#fff,color:#fff
    style Coerce fill:#555,stroke:#fff,color:#fff
    style Store fill:#2a2,stroke:#fff,color:#fff
    style Store2 fill:#2a2,stroke:#fff,color:#fff
    style Store3 fill:#c42,stroke:#fff,color:#fff
```
