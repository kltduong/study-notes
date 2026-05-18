# `this` Determination — Sub-part Checks

Detailed Q&A from sub-part checks during teaching.

---

## Sub-part 1 check: `arr[0]()`

**Question:** What is `this` here?

```js
"use strict";
const obj = { run() { return this; } };
const arr = [obj.run];
arr[0]();
```

**Answer — full Reference/GetValue trace:**

### `[obj.run]` — building the array literal

1. `obj.run` is a member expression:
   - `obj` is an identifier → identifier resolution → `Ref₁ { base: scriptER, name: "obj" }`
   - `.` needs the object → GetValue(`Ref₁`) → follows `scriptER["obj"]` → returns the `obj` object. `Ref₁` consumed.
   - `.run` property access on that object → `Ref₂ { base: obj, name: "run" }`
2. Array element slot needs the value → GetValue(`Ref₂`) → the `run` function object. `Ref₂` consumed.
3. Function object stored at index 0 of the new array.

### `arr[0]()` — the call

1. `arr[0]` is a member expression (computed property access):
   - `arr` is an identifier → identifier resolution → `Ref₃ { base: scriptER, name: "arr" }`
   - `[0]` needs the object → GetValue(`Ref₃`) → the array object. `Ref₃` consumed.
   - `[0]` computed property access on that array → `Ref₄ { base: arr, name: "0" }`
2. Call operator sees `Ref₄`. `[[Base]]` is `arr` (an object) → `thisValue = arr`
3. GetValue(`Ref₄`) → the `run` function object
4. Call the function with `this = arr`

**Key:** `Ref₂ { base: obj }` died during array construction. `Ref₄ { base: arr }` is what the call operator sees — completely independent Reference, different base. `this = arr`.
