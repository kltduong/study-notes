# `const` & immutability — draft

## Plan (teaching order)

- [x] **Chunk opener** — six predictions that surface "const binds bindings, not values"
- [x] **The axiom** — `const` is a *binding-level* constraint, not a *value-level* one
- [x] **What `const` actually does** — spec mechanism (one-flag check on Set; first init only)
- [x] **Value-level immutability** — `Object.freeze`, shallow vs deep, primitive vs object distinction
- [x] **Practical use** — defaults, when `let` is needed, real-world conventions
- [x] **Understanding check** — 2–3 questions

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

> 📝 **Session record (chunk-opener predictions):**
> - **Your answers:** (a) ✅ fine | (b) ✅ fine | (c) ✅ error (didn't name `TypeError`) | (d) ✅ fine | (e) ✅ error | (f) ✅ `1` (assuming L8 skipped) | (g) ✅ binding only
> - **Tightenings:** (c) → `TypeError` specifically (runtime check; one parser exception is `const x;` which is `SyntaxError`). (e) → strict mode throws `TypeError`; sloppy mode silently no-ops — the silent-failure motivation behind strict mode itself.
> - Full reveal table + axiom derivation in the section below — that's the expanded tightening for all six predictions plus (g).

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

> ✅ Locked: (a) TypeError — `name`'s descriptor `[[Writable]]: false`. (b) fine — `roles` slot is locked but the array object is a separate heap allocation, never frozen. (c) TypeError — `roles`'s descriptor `[[Writable]]: false`. (d) TypeError — `user`'s binding has `mutable: false`. Two distinct mechanisms (descriptor flag vs ER flag) firing on the same line family.

---

## Practical use — defaults, when `let` is needed, conventions

The mechanism is one thing; how to *use* it day to day is another. The community has settled on a small set of rules that fall straight out of the model.

### The default rule

> **Use `const` everywhere. Reach for `let` only when you genuinely need to rebind. Never use `var`.**

Three reasons this default works:

1. **Reader signal.** When a maintainer scans `const foo = …`, they know the name will refer to the same value for the rest of the scope. With `let`, they have to scan the whole scope for reassignments. `const` gives a local guarantee that compresses the cognitive load of reading code.
2. **Mistake-catcher.** Accidental rebinding is a real bug class — typos (`user = userData` instead of `user.id = userData.id`), refactors that left a stray assignment. `const` turns those into immediate `TypeError`s instead of silent state corruption.
3. **No runtime cost.** `mutable: false` is one bit. There is zero performance argument for `let` over `const`.

### When `let` is actually needed

`let` is for cases where the *binding* needs to be reassigned. Concretely:

| Pattern                  | Why `let`                                                                  |
| ------------------------ | -------------------------------------------------------------------------- |
| Loop counter             | `for (let i = 0; i < n; i++)` — `i++` is `i = i + 1`, a rebind             |
| Accumulator (mutable)    | `let total = 0; for (...) total += x;`                                     |
| Conditional assignment   | `let value; if (cond) value = a; else value = b;` (though ternary often beats this) |
| Reassignment in branches | `let result = await tryA(); if (!result) result = await tryB();`           |

Notice every case has an **explicit rebind** of the named slot. If you're just mutating an object (`config.foo = 5`), that's not a rebind — `const config = { … }` works.

### When `Object.freeze` is worth it

Frozen objects have a real runtime cost (every mutation hits the descriptor check), so the reach is selective:

- **Module-level config / constants.** `const ROUTES = Object.freeze({ home: "/", about: "/about" })` — frozen once at module load, read constantly.
- **Enum-like objects.** `const Status = Object.freeze({ PENDING: 0, ACTIVE: 1, DONE: 2 })` — semantic intent is "these never change."
- **Defensive copies handed across module boundaries** — when you don't trust callers not to mutate.
- **Test fixtures** — to catch tests that accidentally mutate shared state.

When **not** to freeze:

- Per-request data, working state, anything you'll modify next millisecond.
- Frequently allocated objects (the freeze cost adds up).
- Anywhere a downstream library expects to mutate the object you pass it.

### What about `Object.freeze` deep?

Most codebases don't deep-freeze. The shallow version + naming convention (`SCREAMING_SNAKE_CASE` for module-level constants) is enough signal. Reach for `deepFreeze` only when:

- The object is genuinely meant to be a deep-immutable value (e.g. an enum-like with nested namespacing).
- Tests need to enforce no-mutation across a complex shape.

For real "immutable data structure" work, libraries like Immer or Immutable.js are usually a better answer than hand-rolled `deepFreeze` — they give structural sharing and lazy copies, while `freeze` just throws on writes.

### Beyond `const`: the spectrum

| Tool                             | Constrains                              | Use when                                           |
| -------------------------------- | --------------------------------------- | -------------------------------------------------- |
| `const`                          | Binding (no rebind)                     | Default for **all** local declarations             |
| `Object.freeze` (shallow)        | One object's own props                  | Module config, enum-like objects                   |
| `deepFreeze` (recursive)         | Object + everything reachable           | Test fixtures, deeply-nested constants             |
| TypeScript `readonly` / `Readonly<T>` | **Compile-time** binding/prop guards | Catch mutation at typecheck time, zero runtime cost |
| `Object.defineProperty(... { writable: false })` | Specific properties             | Granular per-property locking (rare in app code)   |
| Immer / Immutable.js             | Structural-sharing immutable updates   | Complex state trees (Redux-style)                  |

The progression: bigger lock → more guarantees → more cost. Start at `const` and move up only when there's a concrete reason.

### Common anti-patterns

- **`let` "just in case I need to reassign later."** YAGNI — change to `let` if and when. Default `const` is the louder signal.
- **`const` + `Object.freeze` everywhere.** Over-freezing is a real cost (every property write hits the descriptor check). Reserve for objects with "constant" semantics.
- **Treating `const` as "this value cannot change."** It's the binding-vs-value confusion. Internalize the axiom: binding only.
- **`var` in modern code.** No legitimate use case once `let`/`const` exist (covered in [`var-quirks.md`](./var-quirks.md)).

---

## End-of-chunk understanding check

3 questions, one at a time.

**Q1.** Predict the output:

```js
"use strict";
const COLORS = Object.freeze({
  primary: "#46c",
  shades: ["#46c", "#358", "#234"]
});

COLORS.primary = "#fff";       // L4
COLORS.shades.push("#123");    // L5
COLORS.shades[0] = "#fff";     // L6

console.log(COLORS.primary);   // L8
console.log(COLORS.shades);    // L9
```

What happens at each of L4, L5, L6? What prints at L8 and L9?

> 📝 **Session record:**
> - **Your answer:** L4 error. L5 fine. L6 fine. L8 (assuming L4 skipped): `"#46c"`. L9 (assuming L4 skipped): `["#fff", "#358", "#234"]`.
> - **Tightening 1 (script-level halt):** Per-line mechanism right, but in strict mode L4's `TypeError` is uncaught — execution stops. L5/L6/L8/L9 are unreachable in the as-written script; the only output is the thrown error. Sloppy mode silently no-ops L4 and execution continues. Strict trades silent corruption for early death — same pattern, observable through *what runs after the failure*.
> - **Tightening 2 (chained trace miss):** Under "assume L4 skipped," L9 should be `["#fff", "#358", "#234", "#123"]` — the L5 `push` appended `"#123"` *before* L6 overwrote index 0. Watch for trace-composition errors when stacking mutations on the same object.

---

**Q2.** True or false (and explain): "Using `const` for everything makes a program more memory-efficient, because the engine can avoid allocating a writable slot for the binding."

> 📝 **Session record:**
> - **Your answer:** False. `const` does not make a program more memory-efficient — only one setting field is changed compared to `let`.
> - **Tightening (JIT aside):** Correct. The binding record exists either way; only the `mutable` flag differs (one bit). There *is* a small JIT-level upside — V8 and similar can sometimes specialize on knowing a `const` won't be reassigned (e.g., inlining a top-level `const PI = 3.14`) — but that's a speculative optimization, not a memory saving. The slot still exists, same shape.

---

**Q3.** Walk through this and predict each line — focus on *why*, not just the verdict:

```js
"use strict";
const items = [];

function addItem(x) {  // L4
  items.push(x);       // L5
}

addItem("a");          // L7
addItem("b");          // L8
items = items.concat(["c"]);  // L9
```

For L5, L7, L8, L9 — what happens, and tied to which mechanism (binding constraint vs property descriptor vs neither)?

> 📝 **Session record:**
> - **Your answer:** `const items` sets `mutable: false` on the ER; binding can't change but the heap object setting is untouched. L5 fine, L7 fine, L8 fine, L9 error.
> - **Tightening 1 (closure mechanism for L5):** The closure resolves `items` via `[[OuterEnv]]` walk (lexical scoping, [`lexical-scoping.md`](./lexical-scoping.md)). Reads the heap reference, then `Array.prototype.push` does an internal `[[Set]]` on the heap object. The ER slot is read, never written.
> - **Tightening 2 (binding follows the binding for L7/L8):** The `mutable: false` flag travels with the *binding*, not the call site. There is one binding for `items` (in module/script scope); a rebind anywhere — inside or outside the function — would be blocked. Rebind vs mutate are spec-level different things regardless of *where* they happen.
> - **Tightening 3 (precise op for L9):** `SetMutableBinding("items", newArr, true)` sees `mutable: false` → `TypeError: Assignment to constant variable.`
