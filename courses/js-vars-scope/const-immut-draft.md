# `const` & immutability — draft

## Plan (teaching order)

- [x] **Chunk opener** — six predictions that surface "const binds bindings, not values"
- [x] **The axiom** — `const` is a *binding-level* constraint, not a *value-level* one
- [x] **What `const` actually does** — spec mechanism (one-flag check on Set; first init only)
- [ ] **Value-level immutability** — `Object.freeze`, shallow vs deep, primitive vs object distinction
- [ ] **Practical use** — defaults, when `let` is needed, real-world conventions
- [ ] **Understanding check** — 2–3 questions

---

## Chunk opener — teaser

```js
const arr = [1, 2, 3];   // L1
arr.push(4);             // L2  — (a) error or fine?
arr[0] = 99;             // L3  — (b) error or fine?
arr = [1, 2, 3];         // L4  — (c) error or fine?

const obj = { name: "Alice" };  // L5
obj.name = "Bob";         // L6  — (d) error or fine?

"use strict";
const frozen = Object.freeze({ a: 1 });   // L7
frozen.a = 99;            // L8  — (e) error or fine? if fine, what does it do?
console.log(frozen.a);    // L9  — (f) what prints?
```

Six predictions. Take the script as one unit (assume strict mode throughout, even though `"use strict"` appears mid-snippet for readability). For each line marked `(x)`:

- **error** (specify which kind: SyntaxError / TypeError / ReferenceError), or
- **fine** (and if it has an observable effect — what is it?)

The "wow" question, save for after the predictions land:

- (g) What is `const` actually constraining — the binding `arr`, the array object it points to, or both?

---

## The reveal — answers + the axiom

Predictions vs reality:

| Line | Code                  | Result (strict)                      | Why                                                                                |
| ---- | --------------------- | ------------------------------------ | ---------------------------------------------------------------------------------- |
| L2   | `arr.push(4)`         | fine — `arr` is now `[1,2,3,4]`      | Mutates the array object. Binding `arr` still points to the same object.           |
| L3   | `arr[0] = 99`         | fine — `arr` is now `[99,2,3,4]`     | Same — property write on the object. No rebind.                                    |
| L4   | `arr = [1,2,3]`       | **TypeError**                        | Rebind attempt — assigns a new value to the binding. `const` blocks this.          |
| L6   | `obj.name = "Bob"`    | fine                                 | Property write. No rebind.                                                         |
| L8   | `frozen.a = 99`       | **TypeError** (strict) / no-op (sloppy) | `Object.freeze` marks every own property non-writable + non-configurable. Strict throws on writes to non-writable; sloppy silently fails. |
| L9   | `console.log(frozen.a)` | `1` (if L8 was skipped or sloppy)  | The write never landed — value unchanged.                                          |

### The axiom

> **`const` is a constraint on the binding, not on the value.**

Restated mechanically:

- **Binding** = the name → value link. The slot in the Environment Record.
- **Value** = whatever sits at the other end of that link (a primitive, or a reference to an object).
- `const` says: *this slot, once initialized, cannot be re-pointed.*
- It says **nothing** about whether the thing the slot points to can be modified.

> **Aside —** What actually sits in the slot differs by value type. For **primitives** (`number`, `string`, `boolean`, `null`, `undefined`, `bigint`, `symbol`), the value itself lives directly in the slot — there's nothing else to point to. For **objects** (arrays, functions, plain objects, etc.), the slot holds a *reference* (a pointer to the heap-allocated object). So `const` on a primitive means the value can never change (rebind is the only operation, and it's blocked). `const` on an object means the pointer is frozen but the heap object behind it is fully mutable — two layers, `const` only locks the first.

This is why L2/L3/L6 all run fine. They reach through the binding to the object and mutate the object. The binding `arr` itself was never touched — it still points to the same array, just an array with different contents.

L4 is different: `arr = [1, 2, 3]` is asking "make `arr` point at a *new* array." That's a rebind. `const` rejects it.

### Why this is the only coherent design

Two reasons fall straight out of the model:

1. **Bindings and values live in different places.** The Environment Record holds the binding (name + slot). The heap holds the object. `const` is a flag on the slot, in the ER. It has no presence on the heap object — so it can't possibly constrain mutation.
2. **Every other declaration form already separates the two.** `let`/`var` rebinding (`x = newVal`) and value mutation (`obj.foo = ...`) are different operations at the spec level — `PutValue` on a Reference vs `[[Set]]` on an object. `const` only blocks the first. The second was never something `const` could touch without a separate mechanism (which is what `Object.freeze` is).

Take a moment with this — anything you want to push back on or test before we go to the spec mechanism?

---

## What `const` actually does — the spec mechanism

The whole behavior comes from **one bit on the binding** in the Environment Record.

### Setup — what the ER stores

A Declarative Environment Record (the kind `let`/`const` use) stores per-binding metadata. The relevant fields:

| Field             | Purpose                                                                    |
| ----------------- | -------------------------------------------------------------------------- |
| `name`            | The identifier (`x`, `arr`, etc.)                                          |
| `value`           | The current value (or "uninitialized" sentinel during TDZ)                 |
| **`mutable`**     | Boolean. `true` for `let`, **`false` for `const`**. The single distinguishing flag. |
| `strict`          | Strict-mode flag (used for some error reporting)                           |

That `mutable: false` flag is **the entirety of `const`'s mechanism**. Everything else — TDZ, scoping, hoisting — is identical between `let` and `const`. They share the same creation-phase logic, the same Block ER routing, the same TDZ window.

### Two abstract operations create the binding

The creation phase calls one of two ER methods, depending on the keyword:

- **`CreateMutableBinding(name)`** for `let` → installs binding with `mutable: true`
- **`CreateImmutableBinding(name)`** for `const` → installs binding with `mutable: false`

Both leave the slot in the "uninitialized" state (TDZ). Initialization is a separate step.

### Two abstract operations write to the binding

- **`InitializeBinding(name, value)`** — sets the slot to `value` for the **first time**, exiting TDZ. Called when control reaches the declarator's initializer (`= 5`). For `const`, this is the **only** way to ever put a value in the slot.
- **`SetMutableBinding(name, value, strict)`** — used for **subsequent** writes (any reassignment). This is the operation behind `x = newVal`.

The `const` enforcement rule lives inside `SetMutableBinding`:

```text
SetMutableBinding(N, V, S):
  if binding for N is uninitialized → throw ReferenceError  (TDZ)
  if binding for N has mutable = false:
      if S (strict)  → throw TypeError
      else           → silently fail   ← legacy sloppy path
  else
      set the slot's value to V
```

That's it. The `if mutable === false` branch is `const`. One conditional, one error.

### Why "first init only"

`const x = 5` is parsed as **declarator + initializer**, not as "declare then assign". At runtime:

1. Creation phase: `CreateImmutableBinding("x")` — slot exists, marked immutable, in TDZ.
2. Execution reaches the line: `InitializeBinding("x", 5)` — first and only legal write.

There's no `SetMutableBinding` in that flow at all — so the immutability check never even runs. That's why `const x = 5` works even though `x` is "immutable" — `Initialize` ≠ `Set`. The two are different operations and the immutability flag only gates the second.

This is also why **`const x;` (no initializer) is a SyntaxError**, not a runtime error. Since `const` only allows one write, and that write must be the initializer, the parser refuses to even accept a `const` declarator without `=`. The grammar enforces it before runtime.

### One-line summary

> `const` = `let` + the binding's `mutable` flag set to `false` + the parser requiring an initializer. Everything else is shared.

### Worked trace — the snippet from the opener

```js
const arr = [1, 2, 3];   // L1
arr.push(4);             // L2
arr = [1, 2, 3];         // L4
```

Mechanism trace:

| Step    | Operation                              | Effect                                                                                                  |
| ------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Creation | `CreateImmutableBinding("arr")`        | ER slot for `arr`: `{ value: <uninit>, mutable: false }`                                                |
| L1 exec  | `InitializeBinding("arr", [1,2,3])`    | First write. Slot now holds reference to the array. **No mutability check.**                            |
| L2 exec  | (no ER op) `arr.push(4)`               | Resolves `arr` → reference. Calls `[[Set]]` on the *array object* (the heap object). ER slot untouched. |
| L4 exec  | `SetMutableBinding("arr", [1,2,3], true)` | Subsequent write. Sees `mutable: false` → **TypeError**.                                              |

L2 doesn't go through the ER's set operation at all — it goes through the object's `[[Set]]`. That's the spec-level reason `const` is structurally incapable of preventing mutation.

---

Quick check before moving on:

**Q1.** Why is `const x;` a SyntaxError rather than a TypeError or ReferenceError? Tie your answer to which abstract operation would or wouldn't run.

> ✅ Locked: `CreateImmutableBinding` would run, `InitializeBinding` never fires (no `=`), `SetMutableBinding` is blocked. The binding would be permanently unusable — every access throws, every write throws. The grammar prunes this dead state statically.

---

## Value-level immutability — `Object.freeze`

If `const` only locks the binding, what locks the value? **`Object.freeze`** — and it's a completely separate mechanism, operating on the heap object instead of the ER.

### What `freeze` does

`Object.freeze(obj)` walks every **own** property of `obj` and:

1. Sets each property's `[[Writable]]` descriptor to `false` → blocks `obj.x = newVal`.
2. Sets each property's `[[Configurable]]` descriptor to `false` → blocks `delete obj.x` and blocks redefining the descriptor.
3. Sets the object's `[[Extensible]]` internal slot to `false` → blocks adding new properties.

After freezing, every write/delete/add hits one of these guards and either silently fails (sloppy) or throws `TypeError` (strict).

### Two layers, two locks

```text
  ┌─────────────────────────────┐
  │   Environment Record        │
  │   ┌─────────────────────┐   │
  │   │ name: "obj"         │   │   ← const locks this slot
  │   │ value: ─────────────┼───┼──► (heap pointer)
  │   │ mutable: false      │   │
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

| Lock           | Operates on        | Stored in                  | Blocks                                |
| -------------- | ------------------ | -------------------------- | ------------------------------------- |
| `const`        | The binding (slot) | Environment Record         | Reassigning the variable (`x = …`)    |
| `Object.freeze` | The object         | The heap object's slots    | Mutating / adding / deleting props    |

The two locks are **orthogonal**. Possible combinations:

| Binding | Object  | Result                                              |
| ------- | ------- | --------------------------------------------------- |
| `let`   | normal  | rebind ✅, mutate ✅                                |
| `let`   | frozen  | rebind ✅, mutate ❌                                |
| `const` | normal  | rebind ❌, mutate ✅                                |
| `const` | frozen  | rebind ❌, mutate ❌ — full immutability            |

The "full immutability" combo requires **both** — neither alone is enough.

### The "shallow" qualifier — what it actually means

`Object.freeze` is documented as **shallow**. Concrete demonstration:

```js
const config = Object.freeze({
  name: "Alice",
  prefs: { theme: "dark" }   // L2 — nested object
});

config.name = "Bob";        // L4 — TypeError (strict). name is frozen.
config.prefs = {};          // L5 — TypeError (strict). prefs slot is frozen.
config.prefs.theme = "light"; // L6 — fine! prefs *object* was never frozen.
console.log(config.prefs.theme); // "light"
```

The reason: `freeze` walks `config`'s own properties and locks each *property descriptor*. The `prefs` descriptor is locked — you can't change what `prefs` points to. But the object that `prefs` points to is a separate heap object, and `freeze` never touched it.

It's the same two-layer pattern as `const` itself, just one level lower: **the slot for `prefs` is locked; the object it points to is not.**

### Deep freeze — opt in by recursion

There's no built-in `Object.deepFreeze`. The standard recipe:

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

Three things to notice:

1. **`Object.isFrozen` check** prevents infinite recursion on cycles (e.g. `obj.self = obj`).
2. **Recurse before freezing self** — the object only needs freezing once, but children must be processed first to avoid running into the parent's now-frozen state.
3. **Doesn't follow non-enumerable / Symbol-keyed / prototype-chain properties.** Production-grade implementations (e.g. lodash, immer) handle these — for everyday config objects the simple version is usually enough.

### Primitives don't need freezing

```js
const x = 5;
x = 6;         // TypeError — rebind blocked by const
// There is no "mutate the value 5" operation.
// Numbers, strings, booleans, etc. have no mutator API.
```

Primitives are **value types** — there's no heap object to mutate, no second layer. `const` alone gives full immutability for primitives. `freeze` only matters for objects.

This is also why `const PI = 3.14159` is the canonical use case — primitive + `const` = fully locked, no extra ceremony.

---

Quick check before moving to practical use:

**Q2.** Given:

```js
const user = Object.freeze({
  name: "Alice",
  roles: ["admin", "editor"]
});

user.name = "Bob";              // (a)
user.roles.push("viewer");      // (b)
user.roles = [];                // (c)
user = { name: "Carol" };       // (d)
```

Strict mode. For each line, predict: error (which type) or fine (and what changes)? Answer all four at once.
