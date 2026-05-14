# `const` & Immutability

**TL;DR** — `const` is a constraint on the **binding** (the slot in the Environment Record), not on the **value**. Mechanically it's a single boolean — `mutable: false` on the binding record — checked when subsequent writes are attempted. To lock the *value* of an object you need a separate mechanism (`Object.freeze`), which sets `[[Writable]]: false` on each property descriptor on the heap object. The two locks are **orthogonal**: full immutability requires both. `freeze` is **shallow** by design — it locks the object's own property slots, not the objects those slots point to.

Terms used: **ER** = Environment Record, **binding** = the `(name, value, mutable)` record an ER stores per identifier (see [execution-context.md](execution-context.md)), **descriptor** = the `{ value, writable, enumerable, configurable }` record an object stores per own property, **`PutValue`** = spec op for `x = v`, **`[[Set]]`** = spec op for `obj.p = v`. `Initialize` vs `Set` distinction relies on the 3-stage lifecycle in [variable-lifecycle.md](variable-lifecycle.md).

---

## 1. The axiom

> **`const` is a constraint on the binding, not on the value.**

Restated mechanically:

- **Binding** = the name → value link. The slot in the ER.
- **Value** = whatever sits at the other end of that link.
- `const` says: *this slot, once initialized, cannot be re-pointed.*
- It says **nothing** about whether the thing the slot points to can be modified.

> **Aside —** What sits in the slot differs by value type. For **primitives** (`number`, `string`, `boolean`, `null`, `undefined`, `bigint`, `symbol`), the value lives directly in the slot — there's nothing else to point to. For **objects**, the slot holds a *reference* to a heap-allocated object. So `const` on a primitive locks everything (rebind is the only operation, and it's blocked). `const` on an object locks the pointer; the heap object behind it is fully mutable.

### Why this is the only coherent design

Two structural reasons fall straight out of the model:

1. **Bindings and values live in different places.** The binding lives in an ER. The object lives on the heap. `const` is a flag on the slot in the ER — it has no presence on the heap object, so it physically cannot constrain mutation.
2. **The spec already separates the two operations.** Rebinding (`x = newVal`) is `PutValue` on a `Reference`. Mutation (`obj.foo = …`) is `[[Set]]` on an object. These are two different abstract operations on two different parts of the runtime. `const` only gates the first; the second was never something `const` could touch without a second mechanism — which is what `Object.freeze` is.

---

## 2. The spec mechanism — one flag, four operations

The whole `const` behavior is **one bit on the binding record**, gated by the difference between two write operations.

### What the ER stores per binding

A Declarative ER (the kind `let`/`const` use) holds, per binding:

| Field         | Purpose                                                                  |
| ------------- | ------------------------------------------------------------------------ |
| `name`        | The identifier                                                           |
| `value`       | Current value, or "uninitialized" sentinel (TDZ — see [tdz.md](tdz.md))  |
| **`mutable`** | Boolean. `true` for `let`, **`false` for `const`**. The single distinguishing flag. |
| `strict`      | Strict-mode flag (used for error-reporting choices)                      |

`mutable: false` is the entirety of `const`'s mechanism. Everything else — TDZ, scoping, hoisting — is identical between `let` and `const`. They share the same creation-phase logic, the same Block ER routing, the same TDZ window. See [creation-execution.md](creation-execution.md) for the per-keyword creation rules.

### Two creation operations

The creation phase calls one of two ER methods, depending on the keyword:

- **`CreateMutableBinding(name)`** for `let` → installs the binding with `mutable: true`.
- **`CreateImmutableBinding(name)`** for `const` → installs the binding with `mutable: false`.

Both leave the slot in the "uninitialized" state (TDZ).

### Two write operations

- **`InitializeBinding(name, value)`** — sets the slot to `value` for the **first time**, exiting TDZ. Called when control reaches the declarator's initializer (`= 5`). For `const`, this is the **only** way to ever put a value in the slot. **No mutability check runs here.**
- **`SetMutableBinding(name, value, strict)`** — used for **subsequent** writes (any reassignment). The `const` enforcement lives inside this op.

```text
SetMutableBinding(N, V, S):
  if binding for N is uninitialized → throw ReferenceError  (TDZ)
  if binding for N has mutable = false:
      if S (strict)  → throw TypeError
      else           → silently fail   ← legacy sloppy path
  else
      set the slot's value to V
```

That's `const`'s entire enforcement: one branch in `SetMutableBinding`.

### Why `const x = 5` works even though `x` is "immutable"

Because `Initialize` ≠ `Set`. The mutability check is only inside `SetMutableBinding`. The initializer runs `InitializeBinding`, which has no such check. They are **two different abstract operations** on the same slot — `const` only gates the second.

### Why `const x;` is a SyntaxError, not a runtime error

If `const x;` were allowed:

- `CreateImmutableBinding("x")` would run at creation phase — slot exists, `mutable: false`, in TDZ.
- `InitializeBinding` would never fire (no `= …`), so the slot would stay in TDZ forever.
- `SetMutableBinding` is also blocked (any later `x = 5` sees `mutable: false` → TypeError).
- Net effect: a permanently unusable binding — every read throws `ReferenceError`, every write throws `TypeError`.

The grammar prunes this dead state statically rather than letting it create a useless binding at runtime. Static enforcement of a runtime impossibility.

### One-line summary

> `const` = `let` + the binding's `mutable` flag set to `false` + the parser requiring an initializer. Everything else is shared.

---

## 3. Object.freeze — locking the value

If `const` only locks the binding, what locks the value? **`Object.freeze`** — and it's a completely separate mechanism, operating on the heap object instead of the ER.

### What freeze does

`Object.freeze(obj)` walks every **own** property of `obj` and:

1. Sets each property's `[[Writable]]` descriptor field to `false` → blocks `obj.x = newVal`.
2. Sets each property's `[[Configurable]]` field to `false` → blocks `delete obj.x` and blocks redefining the descriptor.
3. Sets the object's `[[Extensible]]` internal slot to `false` → blocks adding new properties.

After freezing, every write/delete/add hits one of these guards and either silently fails (sloppy) or throws `TypeError` (strict).

### Two layers, two locks

```text
  ┌─────────────────────────────┐
  │   Environment Record        │
  │   ┌─────────────────────┐   │
  │   │ name: "obj"         │   │
  │   │ value: ─────────────┼───┼──► (heap pointer)
  │   │ mutable: false      │   │   ← const locks this slot
  │   └─────────────────────┘   │
  └─────────────────────────────┘
                  │
                  ▼
  ┌─────────────────────────────┐
  │   Heap object               │
  │   { a: 1, b: 2 }            │   ← Object.freeze locks this
  │   [[Extensible]]: false     │
  │   own props: writable=false │
  └─────────────────────────────┘
```

| Lock           | Operates on             | Stored in            | Blocks                                |
| -------------- | ----------------------- | -------------------- | ------------------------------------- |
| `const`        | The binding (slot)      | Environment Record   | Reassigning the variable (`x = …`)    |
| `Object.freeze` | The object              | Heap object's slots  | Mutating / adding / deleting props    |

The two are **orthogonal**:

| Binding | Object  | Result                                          |
| ------- | ------- | ----------------------------------------------- |
| `let`   | normal  | rebind ✅, mutate ✅                            |
| `let`   | frozen  | rebind ✅, mutate ❌                            |
| `const` | normal  | rebind ❌, mutate ✅                            |
| `const` | frozen  | rebind ❌, mutate ❌ — full immutability        |

Full immutability needs **both** — neither alone is enough.

### Shallow by design

`Object.freeze` only walks **own** properties. Nested objects sit one indirection deeper and are untouched.

```js
const config = Object.freeze({
  name: "Alice",
  prefs: { theme: "dark" }     // L2 — nested object
});

config.name = "Bob";            // L4 — TypeError (strict). name is frozen.
config.prefs = {};              // L5 — TypeError (strict). prefs slot is frozen.
config.prefs.theme = "light";   // L6 — fine! prefs *object* was never frozen.
console.log(config.prefs.theme); // "light"
```

Same two-layer pattern as `const` itself, just one level lower: the slot for `prefs` is locked; the object it points to is not. The pattern recurses.

### Deep freeze — opt in by recursion

There is no built-in `Object.deepFreeze`. The standard recipe:

```js
function deepFreeze(obj) {
  Object.values(obj).forEach(v => {
    if (v !== null && typeof v === "object" && !Object.isFrozen(v)) {
      deepFreeze(v);
    }
  });
  return Object.freeze(obj);
}
```

Three subtleties:

1. **`Object.isFrozen` check** prevents infinite recursion on cycles (`obj.self = obj`).
2. **Recurse before freezing self** — children must be processed first; once a parent is frozen, you can still read its descendants but the structure is sealed.
3. **Doesn't follow non-enumerable / Symbol-keyed / prototype-chain properties.** Production-grade implementations handle these; for everyday config the simple version is enough.

### Primitives don't need freezing

```js
const x = 5;
x = 6;         // TypeError — rebind blocked by const
// There is no "mutate the value 5" operation — primitives have no mutator API.
```

`const` alone gives full immutability for primitives. `freeze` only matters for objects. This is why `const PI = 3.14159` is the canonical use case — primitive + `const` = fully locked, no extra ceremony.

---

## 4. Practical defaults

The community has settled on a small set of rules that fall straight out of the model.

### The default rule

> **Use `const` everywhere. Reach for `let` only when you genuinely need to rebind. Never use `var`** (covered in [var-quirks.md](var-quirks.md)).

Three reasons this default works:

1. **Reader signal.** When a maintainer scans `const foo = …`, they know the name will refer to the same value for the rest of the scope. With `let`, they have to scan the whole scope for reassignments. `const` gives a local guarantee that compresses the cognitive load of reading code.
2. **Mistake-catcher.** Accidental rebinding is a real bug class — typos (`user = userData` instead of `user.id = userData.id`), refactors that left a stray assignment. `const` turns those into immediate `TypeError`s instead of silent state corruption.
3. **No runtime cost.** `mutable: false` is one bit. There is zero performance argument for `let` over `const`.

### When `let` is actually needed

`let` is for cases where the *binding* needs to be reassigned. Concretely:

| Pattern                  | Why `let`                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------- |
| Loop counter             | `for (let i = 0; i < n; i++)` — `i++` is `i = i + 1`, a rebind                         |
| Accumulator              | `let total = 0; for (...) total += x;`                                                |
| Conditional assignment   | `let value; if (cond) value = a; else value = b;` (ternary often beats this)          |
| Reassignment in branches | `let result = await tryA(); if (!result) result = await tryB();`                      |

If you're only mutating an object (`config.foo = 5`), that's not a rebind — `const config = { … }` works.

### When `Object.freeze` is worth it

Frozen objects have a real runtime cost (every mutation hits the descriptor check), so the reach is selective:

- **Module-level config / constants** — frozen once at load, read constantly.
- **Enum-like objects** — `const Status = Object.freeze({ PENDING: 0, ACTIVE: 1, DONE: 2 })`.
- **Defensive copies handed across module boundaries** — when you don't trust callers.
- **Test fixtures** — to catch tests that accidentally mutate shared state.

When **not** to freeze:

- Per-request / working-state data you'll modify next millisecond.
- Frequently allocated objects (the freeze cost adds up).
- Anywhere a downstream library expects to mutate the object.

### The lock spectrum

| Tool                                              | Constrains                                | Use when                                   |
| ------------------------------------------------- | ----------------------------------------- | ------------------------------------------ |
| `const`                                           | Binding (no rebind)                       | Default for **all** local declarations     |
| `Object.freeze` (shallow)                         | One object's own props                    | Module config, enum-like objects           |
| `deepFreeze` (recursive)                          | Object + everything reachable             | Test fixtures, deeply-nested constants     |
| TypeScript `readonly` / `Readonly<T>` / `as const` | **Compile-time** binding/prop guards     | Catch mutation at typecheck, zero runtime cost |
| `Object.defineProperty(... { writable: false })`  | Specific properties                       | Granular per-property locking (rare)       |
| Immer / Immutable.js                              | Structural-sharing immutable updates      | Complex state trees (Redux-style)          |

Bigger lock → more guarantees → more cost. Start at `const` and move up only when there's a concrete reason.

### Common anti-patterns

- **`let` "just in case I need to reassign later."** YAGNI — change to `let` if and when. Default `const` is the louder signal.
- **`const` + `Object.freeze` everywhere.** Over-freezing has real cost. Reserve for objects with "constant" semantics.
- **Treating `const` as "this value cannot change."** The binding-vs-value confusion. Internalize the axiom: binding only.

---

## 5. Worked synthesis — the two locks across a closure

```js
"use strict";

const items = [];                // L1 — const binding; ER slot 'items' is mutable=false; value = ref to []

function addItem(x) {            // L3
  items.push(x);                 // L4 — closure walks [[OuterEnv]], finds 'items' binding,
                                 //       reads the heap reference, calls Array.prototype.push
                                 //       which does internal [[Set]] on the heap array.
                                 //       The ER slot is read, never written. ✅ allowed.
}

addItem("a");                    // L7 — new function EC; same mechanism as L4. items now ["a"]
addItem("b");                    // L8 — same. items now ["a","b"]

items = items.concat(["c"]);     // L10 — concat returns a NEW array.
                                 //        '= …' attempts SetMutableBinding("items", newArr, true).
                                 //        sees mutable: false → TypeError.

const cfg = Object.freeze({ list: items });  // L13 — freezes cfg's own props (list is now non-writable).
                                              //        items array itself is unchanged — separate heap object.

cfg.list = [];                   // L15 — TypeError. cfg.list's descriptor [[Writable]] is false.
cfg.list.push("d");              // L16 — fine. The array cfg.list points to was never frozen.
console.log(cfg.list);           // ["a", "b", "d"]
```

What this trace shows in one sweep:

- **L4 vs L10** — the binding-vs-value axiom across a closure boundary. The `const` flag travels with the *binding*, not the call site. Mutating the heap object (L4) goes through `[[Set]]` on the array; rebinding (L10) goes through `SetMutableBinding` on the ER slot. Two different ops, only one is gated.
- **L15 vs L16** — the same axiom one level deeper, this time enforced by `freeze`'s descriptor flag instead of `const`'s ER flag. Same shape, different mechanism: the slot is locked, the object the slot points to is not.
- **L13's freeze is shallow** — only `cfg`'s own properties (`list`) get `[[Writable]]: false`. The array sitting at `cfg.list` is a separate heap object with its own (untouched) descriptors. The pattern recurses indefinitely — every layer is a separate lock decision.

---

## 6. Takeaway

- **The axiom:** `const` constrains the binding (the slot in the ER), not the value. The slot can no longer be re-pointed; the heap object behind it is fully mutable.
- **The mechanism:** one boolean (`mutable: false`) on the binding record, checked inside `SetMutableBinding`. The initializer goes through `InitializeBinding`, which has no such check — that's why `const x = 5` works.
- **`const x;` is a SyntaxError, not a runtime error**, because the resulting binding would be permanently unusable. The grammar prunes the dead state statically.
- **`Object.freeze` is the value-level counterpart** — operates on the heap object via `[[Writable]]`/`[[Configurable]]` descriptors and `[[Extensible]]`. Shallow by design — only own properties; nested objects need recursion.
- **The two locks are orthogonal.** Full immutability requires both. `let` + frozen, `const` + unfrozen, etc. are all valid points in the matrix.
- **Default: `const` first; downgrade to `let` only on explicit rebind; reach for `freeze` selectively.** TypeScript `readonly` gives compile-time locking with zero runtime cost when the project supports it.
