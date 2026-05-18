# `this` Determination — Draft

## Plan (teaching order)

- [x] Sub-part 1: The single rule formalized — EvaluateCall reads `[[Base]]`, passes it as thisValue to the new Function EC
- [ ] Sub-part 2: Where `[[ThisValue]]` lands — the Function ER slot, accessible via `this` keyword
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

- L2: `Ref { base: obj }` → `this = obj` → `"Alice"`
- L3: `Ref { base: obj.nested }` → `this = obj.nested` → `"Bob"`
- L4 — two call sites, trace each:
  1. **Outer call:** `callWith(obj.greet)` — `obj.greet` is an argument expression. Argument passing calls GetValue on `Ref { base: obj, name: "greet" }`, extracts the function object, and copies it into `callWith`'s parameter slot `fn`. The Reference (and its `base: obj`) is gone.
  2. **Inner call:** `fn()` inside `callWith` — `fn` is a plain identifier. Identifier resolution finds it in `callWith`'s Function ER → produces `Ref { base: callWith's ER, name: "fn" }`. ER base → `this = undefined` → `this.name` → TypeError.

  The function object is the same `greet` — but the Reference that reaches the inner call operator has an ER base, not `obj`.

- L5 — destructuring is assignment:
  1. **Destructuring:** `const { greet } = obj` — under the hood this evaluates `obj.greet` (producing `Ref { base: obj, name: "greet" }`), then assigns the result to the local binding `greet`. Assignment calls GetValue → extracts the function object → stores it in the module/script ER slot `greet`. Reference consumed.
  2. **Call:** `greet()` — plain identifier. Identifier resolution finds `greet` in the enclosing ER → produces `Ref { base: global/script ER, name: "greet" }`. ER base → `this = undefined` → `this.name` → TypeError.

  Same mechanism as L4 — any operation that extracts the function object from its original member-expression context (argument passing, assignment, destructuring, `return`) strips the Reference. The subsequent call resolves the identifier fresh, gets an ER base, and `this` is `undefined`.

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


