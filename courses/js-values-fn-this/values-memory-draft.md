# Values & Memory Model — Draft

## Plan (teaching order)

- [x] The two value categories and what a slot holds
- [x] Copy semantics: the one rule
- [x] Identity (`===`) as slot-content comparison
- [x] Implications for mutation, aliasing, and argument passing
- [x] Connection forward: why this matters for `this`

---

## The two value categories and what a slot holds

### Axiom: one storage rule, two value categories

Every variable binding in JS is a **slot** — a cell in an Environment Record. The engine stores exactly one thing in each slot. What that thing *is* depends on the value category:

| Category    | What the slot holds                        | Examples                              |
| ----------- | ------------------------------------------ | ------------------------------------- |
| **Primitive** | The value itself                           | `1`, `"hi"`, `true`, `null`, `undefined`, `42n`, `Symbol()` |
| **Object**    | A reference (opaque handle) to a heap object | `{}`, `[]`, `function(){}`, `new Date()` |

There is no third option. Every JS value falls into one of these two categories.

### Why "reference" and not "pointer"

In C, a pointer is a numeric address you can inspect, increment, cast. In JS, the reference is:

- **Opaque** — you can't see or manipulate the address.
- **Auto-dereferenced** — `obj.x` follows the reference implicitly; there's no `*obj` syntax.
- **Not a value you can store separately from the object it points to** — you can't have a "reference to a reference."

The mental model: a slot holding an object value contains an invisible arrow to a heap-allocated structure. You interact with the structure through the arrow, but you never touch the arrow itself.

### The heap vs the slot

```
┌────────────────────────────────────────────┐
│ Environment Record (slots)                 │
├──────────┬─────────────────────────────────┤
│  a       │  ●───────────────┐              │
│  b       │  ●───────────┐   │              │
│  c       │  1           │   │              │
│  d       │  1           │   │              │
└──────────┴─────────────────────────────────┘
                          │   │
                          ▼   ▼
                    ┌──────────────┐
                    │ { x: 1 }     │  ← heap object
                    └──────────────┘
```

After `let b = a`: both slots hold a copy of the same reference → same arrow → same heap object.
After `let d = c`: both slots hold independent copies of the value `1`.

### The seven primitives (exhaustive)

`number`, `string`, `boolean`, `undefined`, `null`, `bigint`, `symbol`. Everything else — including things that *feel* primitive like `new Number(3)` — is an object.

> **Aside — `typeof null === "object"`.** This is a historical bug from the first JS implementation (type tag was stored in the lowest bits; `null` was the null pointer `0x00`, which shared the object tag). It's arbitrary — memorize it, don't derive it.


---

## Copy semantics: the one rule

### The axiom

> **Assignment copies the slot's content. Always. Unconditionally.**

This is the *only* rule. There is no "pass by reference" mode, no special object-assignment behavior. The apparent difference between primitives and objects is a consequence of *what the slot contains*, not a different copying mechanism.

| Situation | What's in the slot | What gets copied | Effect |
|---|---|---|---|
| `let b = a` (a holds primitive) | The value itself | The value | Independent copy — no aliasing |
| `let b = a` (a holds object) | A reference | The reference | Both slots point to same heap object — aliasing |
| `fn(a)` (a holds primitive) | The value itself | The value into parameter slot | Function can't affect caller's slot |
| `fn(a)` (a holds object) | A reference | The reference into parameter slot | Function can mutate the heap object; can't reassign caller's slot |

### Argument passing is assignment

Function parameters are local slots in the function's Environment Record. Calling `fn(a)` copies `a`'s slot content into the parameter slot — same rule as `let param = a`. No special "pass by" semantics.

```js
function replace(obj) {   // L1 — obj slot gets a copy of the reference
  obj = { x: 99 };       // L2 — overwrites the LOCAL slot; caller unaffected
}

function mutate(obj) {    // L3 — obj slot gets a copy of the reference
  obj.x = 99;            // L4 — follows the reference to the heap object; mutates it
}

let thing = { x: 1 };    // L5
replace(thing);           // L6
console.log(thing.x);    // L7 → 1 (replace overwrote its own slot, not ours)

mutate(thing);            // L8
console.log(thing.x);    // L9 → 99 (mutate followed the shared reference)
```

The distinction: **reassigning a slot** (`obj = ...`) only affects that slot. **Mutating through a reference** (`obj.x = ...`) affects the shared heap object visible from all slots holding that reference.

### Why "pass by sharing" (not "pass by reference")

Some languages have true pass-by-reference — the callee can reassign the caller's variable (C++ `&`, C# `ref`). JS can't do that. The callee gets its own slot containing a *copy* of the reference. Reassigning the callee's slot doesn't touch the caller's slot.

The accurate term is **"pass by sharing"** (or "call by object-sharing") — coined by Barbara Liskov for CLU, and it's exactly what JS, Python, Java, and Ruby do.


---

## Identity (`===`) as slot-content comparison

### The rule

> **`===` compares the raw content of two slots. Nothing more.**

No deep inspection, no structural comparison. It asks: "are these two slot values bit-for-bit identical?"

| Category | What's compared | Consequence |
|---|---|---|
| Primitive | The value itself | `1 === 1` → `true` (same value in both slots) |
| Object | The reference | `{} === {}` → `false` (two different references, even if structures are identical) |

### Derivation from the storage model

This isn't a separate rule to memorize — it *falls out* of the slot model:

- Primitives: slot holds the value → comparing slots compares values → structurally identical primitives are `===`.
- Objects: slot holds a reference → comparing slots compares references → two objects with identical structure are still `!==` because they're different heap allocations with different references.

```js
const a = { x: 1 };   // L1 — fresh heap object, reference stored in a's slot
const b = { x: 1 };   // L2 — different heap object, different reference
const c = a;           // L3 — copies a's reference into c's slot

a === b;  // L4 → false (different references, despite identical structure)
a === c;  // L5 → true  (same reference — both slots point to same heap object)
```

### `NaN` — the one exception

`NaN === NaN` is `false`. This is IEEE 754 behavior (NaN is "not a number" — it represents an undefined result, and no undefined result equals another). It's arbitrary from a JS perspective — memorize it. Use `Number.isNaN()` or `Object.is()` when you need to detect NaN.

> **Aside — `Object.is()` vs `===`.** `Object.is` is "same-value" comparison: it fixes the two quirks of `===` (`NaN !== NaN` and `+0 === -0`). For everything else, it behaves identically to `===`. Think of `===` as the fast path that punts on two IEEE 754 edge cases.


---

## Implications for mutation, aliasing, and argument passing

### Aliasing — when it exists and when it doesn't

**Aliasing** = two slots holding the same reference → mutations through one are visible through the other.

- Primitives: **never aliased.** Each slot holds an independent value. You can't mutate a primitive in place (they're immutable), so even if two slots hold `1`, there's no observable sharing.
- Objects: **aliased whenever you copy a reference** (assignment, parameter passing, storing in a property/array). The heap object is shared; any slot holding that reference can mutate it.

### The two operations people confuse

| Operation | Syntax | Effect |
|---|---|---|
| **Reassign the slot** | `x = newValue` | Overwrites this slot only. Other slots holding the old reference are unaffected. |
| **Mutate through the reference** | `x.prop = val`, `x.push(...)` | Follows the reference to the heap object and changes it. All aliases see the change. |

The swap example demonstrated this: `a = b` inside a function reassigns the local slot — the caller's slot is a different cell entirely.

### Primitives are immutable — no mutation path exists

You can't do `"hello".x = 1` and have it stick. String/number/boolean methods return *new* values; they never mutate in place. This is why aliasing is irrelevant for primitives — even if two slots hold the "same" value, there's no operation that could make a change visible through both.

> **Aside — wrapper objects.** `new String("hi")` creates a mutable *object* on the heap. It's not a primitive. `"hi"` (no `new`) is a primitive. When you call `"hi".toUpperCase()`, the engine briefly wraps the primitive in a temporary object, calls the method, discards the wrapper. The primitive in your slot is never touched.


---

## Connection forward: why this matters for `this`

The `[[ThisValue]]` slot in a Function Environment Record holds a value — and that value follows the same storage rules:

- If `this` is an object (the common case), the slot holds a **reference** to that object.
- If `this` is a primitive (rare, but possible in sloppy mode or with `call`/`apply`), the slot holds the primitive directly (or gets coerced to an object in sloppy mode).

Understanding that `this` is just a value in a slot — subject to the same copy/reference semantics as any other binding — demystifies what happens when you "lose `this`." Extracting a method (`const fn = obj.method`) copies the *function reference* into `fn`'s slot, but the `this`-binding isn't part of the function object — it's determined at the call site. The next chunk (Reference type) explains exactly how.

