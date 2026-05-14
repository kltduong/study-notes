# `const` & immutability — draft

## Plan (teaching order)

- [x] **Chunk opener** — six predictions that surface "const binds bindings, not values"
- [ ] **The axiom** — `const` is a *binding-level* constraint, not a *value-level* one
- [ ] **What `const` actually does** — spec mechanism (one-flag check on Set; first init only)
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
