# Values & Memory Model

**TL;DR:** Every binding is a slot. Primitives store the value directly in the slot; objects store an opaque reference to a heap-allocated structure. Assignment always copies slot content — the apparent "copy vs share" difference is a consequence of what's in the slot, not a different mechanism. `===` compares slot content directly.

## Two value categories, one storage rule

| Category      | What the slot holds                          | Examples                                                     |
| ------------- | -------------------------------------------- | ------------------------------------------------------------ |
| **Primitive** | The value itself                             | `number`, `string`, `boolean`, `undefined`, `null`, `bigint`, `symbol` |
| **Object**    | A reference (opaque handle) to a heap object | `{}`, `[]`, `function(){}`, `new Date()`                     |

**Why the split exists** — two constraints force it:

1. **Fixed slot size.** ER slots are uniform cells. Objects are variable-size and can grow at runtime → must live on the heap with a fixed-size reference in the slot.
2. **O(1) assignment.** Copying a slot must be cheap regardless of object size → copy the handle, not the structure.

Primitives satisfy both naturally (small, fixed, immutable) so they live directly in the slot.

### Reference ≠ pointer

JS references are **opaque** (no address arithmetic), **auto-dereferenced** (`obj.x` follows the reference implicitly), and **not composable** (no reference-to-a-reference). Mental model: an invisible arrow from the slot to a heap structure.

```
┌───────────────────────────────────┐
│ Environment Record (slots)        │
├──────────┬────────────────────────┤
│  a       │  ●──────────┐          │
│  b       │  ●──────┐   │          │
│  c       │  1      │   │          │
│  d       │  1      │   │          │
└──────────┴────────────────────────┘
                     │   │
                     ▼   ▼
               ┌──────────────┐
               │ { x: 1 }     │  ← heap object
               └──────────────┘
```

> **Aside — `typeof null === "object"`.** Historical bug from the first JS implementation (type tag stored in lowest bits; `null` was the null pointer `0x00`, sharing the object tag). Arbitrary — memorize it, don't derive it.

## Copy semantics: the one rule

> **Assignment copies the slot's content. Always. Unconditionally.**

No "pass by reference" mode. The apparent difference between primitives and objects is what gets copied (value vs reference), not a different copying mechanism.

| Operation | Syntax | Effect |
|---|---|---|
| **Reassign the slot** | `x = newValue` | Overwrites this slot only. Other slots holding the old reference are unaffected. |
| **Mutate through the reference** | `x.prop = val`, `x.push(...)` | Follows the reference to the heap object. All aliases see the change. |

### Argument passing is assignment

Parameters are local slots. `fn(a)` copies `a`'s slot content into the parameter slot — same rule as `let param = a`.

```js
function replace(obj) {   // L1 — obj slot gets a copy of the reference
  obj = { x: 99 };       // L2 — overwrites LOCAL slot; caller unaffected
}

function mutate(obj) {    // L3 — obj slot gets a copy of the reference
  obj.x = 99;            // L4 — follows reference to heap object; mutates it
}

let thing = { x: 1 };    // L5
replace(thing);           // L6
console.log(thing.x);    // L7 → 1

mutate(thing);            // L8
console.log(thing.x);    // L9 → 99
```

The term is **"pass by sharing"** (Liskov, CLU) — the callee gets its own slot containing a copy of the reference. Reassigning the callee's slot can't touch the caller's. Same model as Python, Java, Ruby.

## Identity (`===`) as slot-content comparison

> **`===` compares the raw content of two slots. Nothing more.**

This falls out of the storage model — not a separate rule:

- Primitives: slot holds value → comparing slots compares values → `"hello" === "hello"` is `true`.
- Objects: slot holds reference → comparing slots compares references → `[] === []` is `false` (different heap allocations, different references).

```js
const a = { x: 1 };   // fresh heap object
const b = { x: 1 };   // different heap object
const c = a;           // copies a's reference

a === b;  // false — different references
a === c;  // true  — same reference
```

> **Aside — `NaN` and `Object.is`.** `NaN === NaN` is `false` (IEEE 754 behavior — arbitrary, memorize it). `Object.is` fixes this plus the `+0 === -0` quirk; otherwise identical to `===`.

## Aliasing

**Aliasing** = two slots holding the same reference → mutations through one visible through the other.

- Primitives: **never aliased.** Immutable values — even if two slots hold `1`, no operation can make a change visible through both.
- Objects: **aliased whenever you copy a reference** (assignment, parameter passing, storing in a property/array).

## Primitives are immutable

No mutation path exists for primitives. `"hi".toUpperCase()` returns a new string; the original is untouched. `+=` on a string creates a new value and reassigns the slot — it doesn't mutate the old string.

> **Aside — wrapper objects.** `new String("hi")` creates a mutable *object* on the heap (not a primitive). When you call `"hi".toUpperCase()`, the engine briefly wraps the primitive in a temporary object, calls the method, discards the wrapper. The primitive in your slot is never touched.

## Forward connection: why this matters for `this`

The `[[ThisValue]]` slot in a Function Environment Record holds a value subject to the same storage rules. When you "lose `this`" by extracting a method (`const fn = obj.method`), you copy the *function reference* into `fn`'s slot — but `this`-binding isn't part of the function object. It's determined at the call site by the Reference type (next chunk).

