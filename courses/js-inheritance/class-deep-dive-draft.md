# Class Deep Dive — Draft

## Plan (teaching order)

- [x] 1. **Unified dispatch rule** — `this` from receiver (recap + forward link); `super.method()` as lookup-redirect with `this` preserved. Worked synthesis: A/B/C cascade.
- [ ] 2. **Polymorphism payoff** — inherited body's `this.method()` resolves to most-derived override because `this` never rebinds.
- [ ] 3. **`super(...)` in derived constructors** — full uninitialized-`this` mechanics, unconditional ReferenceError.
- [ ] 4. **Static dispatch + constructor-side chain** — `this` inside static = class on left of dot; `super.staticMethod()` on constructor chain.
- [ ] 5. **Class fields vs prototype methods (inheritance lens)** — brief recap; inheritance implications.
- [ ] 6. **Private fields (`#field`)** — class-only syntax, spec-enforced encapsulation.
- [ ] 7. **Where "just sugar" leaks** — no-`new` TypeError, TDZ on class binding, `new.target`.
- [ ] 8. **Extending built-ins** — `extends Array/Error/Map`, `Symbol.species`.

---

## Unified dispatch rule

### Teaser

```js
class A {
  hi()    { return 'A.hi'; }
  greet() { return this.hi() + '!'; }
}
class B extends A {
  hi()    { return 'B.hi'; }
}
class C extends B {
  greet() { return 'C says ' + super.greet(); }
}

new C().greet();   // ?
```

**Predict:** what string comes back, and — more importantly — for each method call inside the evaluation, **which class's body runs** and **what is `this`**?

Try to be precise about both: `c.greet()` → ? body, `this` = ?; `super.greet()` → ? body, `this` = ?; `this.hi()` inside that → ? body, `this` = ?

### The axiom

A method call resolves **two independent questions**:

1. **Which function body runs?** — determined by a prototype-chain walk.
2. **What is `this` inside that body?** — always the receiver (the object left of the dot at the original call site).

#### Question 1: which function body runs?

`expr.method()` (normal) and `super.method()` both resolve the function the same way — walk a `[[Prototype]]` chain until the property is found. The only difference is where the walk starts:

| Call form | Walk starts at | Determined when |
|---|---|---|
| `expr.method()` | `expr` itself (the receiver) | call time (dynamic) |
| `super.method()` | `Object.getPrototypeOf(HomeObject)` — one level above where the calling method is *defined* | definition time (static) |

**Normal dispatch** is receiver-based — already derived in [`prototype-deep-dive.md`](prototype-deep-dive.md#custom-prototypes-and-this-binding); not re-derived here.

**Super dispatch** shifts the starting point. `super` is not an object you can store or pass — it's a **lookup instruction** meaning "skip my level." Concretely: start the walk at `Object.getPrototypeOf(HomeObject)`, where `HomeObject` is the object the calling method was *defined on* (the prototype object it was installed into). This start point is baked in at definition time — it never changes regardless of how the method is later called.

```
super.X  ≈  Object.getPrototypeOf(HomeObject)[X]
                        ▲
                 "one above where I'm written"
                 fixed at definition time
```

#### Question 2: what is `this`?

Always the receiver. `super` does not rebind `this` — the parent method runs with the same `this` the caller had. This is what makes inheritance useful: a parent method can reference `this.prop` and get the child instance's data.

```js
d.speak()          // this = d
  → super.speak() // this = still d (Animal.prototype.speak sees d)
```

#### The full desugaring

Putting Q1 and Q2 together into one pseudo-expression:

```
super.X(args)  ≈  Object.getPrototypeOf(HomeObject)[X].call(this, ...args)
                              ▲                                 ▲
                       Q1: where to find the             Q2: receiver,
                       function body (static)            unchanged (dynamic)
```

Two inputs, two different lifetimes: the lookup target is frozen at definition time; `this` flows in at call time. This separation is the entire mechanism — everything else (polymorphism, constructor chaining, the proof below) falls out of it.

#### `[[HomeObject]]` — what anchors `super`

`[[HomeObject]]` is an internal slot on the **function object itself**, set once when the method is defined with **method syntax** inside a class body or object literal. It records *which object this method was installed into* — that's the reference point `super` uses to compute "one level above me."

Key properties:

- **Set once, never changes.** Moving the function to another object doesn't update `[[HomeObject]]`.
- **Only method syntax gets it.** `{ greet() {} }` → has `[[HomeObject]]`. `{ greet: function() {} }` → does not. Arrow functions → do not. This is why `super` is only legal inside method-syntax definitions.
- **Travels with the function.** Detaching a method (`const f = c.greet`) breaks `this` (no receiver at call site) but `super` inside `f` still resolves from the original `[[HomeObject]]`.

The two failure modes are independent because they're sourced from different times:

| Concern | Determined at | Breaks when |
|---|---|---|
| `this` | call time | no receiver (detached call, bare invocation) |
| `super`'s start point | definition time | never breaks once set (but absent if not method syntax) |

##### Demo — detach breaks `this`, not `super`

```js
class A {
  id() { return 'A'; }
}
class B extends A {
  id() { return 'B(super=' + super.id() + ')'; }  // [[HomeObject]] = B.prototype
}

const b = new B();
b.id();              // "B(super=A)" — this=b, super starts at A.prototype ✓

const detached = b.id;
detached();          // TypeError or "B(super=A)" with undefined `this`
                     // super.id() still resolves A.id — HomeObject unchanged
                     // but `this` is undefined (strict mode) — no receiver
```

`super` didn't break — it still found `A.id`. What broke is `this`: no object left of the dot means no receiver. Two independent failure axes.

### Why it *must* be HomeObject-based (not receiver-based)

Proof by contradiction. Suppose `super.X` started its lookup from the parent of **`this`'s class** (receiver-based):

```js
class A { greet() { return 'A→' + (super.greet?.() ?? 'end'); } }  // L1
class B extends A {}                                                // L2 — no greet
class C extends B {}                                                // L3 — no greet
new C().greet();                                                    // A.greet runs, this = c
```

Inside `A.greet`, `this` is `c` (class `C`, parent `B`). Receiver-based reading: look up `greet` from `B.prototype` → not found → `A.prototype` → **finds `A.greet` again** → calls it → `super.greet` again → `B.prototype` → `A.prototype` → `A.greet` → **infinite recursion**.

Calling `super` from an inherited method is legal, everyday code. A rule that makes it infinitely recurse cannot be the rule. Therefore `super`'s start point must be **static** (`HomeObject.[[Prototype]]`), not receiver-derived. Under the real rule: `A.greet`'s `[[HomeObject]]` = `A.prototype`, so `super.greet` starts at `A.prototype.[[Prototype]]` = `Object.prototype` → no `greet` → terminates. No loop. The static rule is the *only* coherent choice.

### Worked synthesis — the A/B/C cascade

```js
class A {
  hi()    { return 'A.hi'; }
  greet() { return this.hi() + '!'; }       // [[HomeObject]] = A.prototype
}
class B extends A {
  hi()    { return 'B.hi'; }                // [[HomeObject]] = B.prototype
}
class C extends B {
  greet() { return 'C says ' + super.greet(); }  // [[HomeObject]] = C.prototype
}

const c = new C();
c.greet();   // 'C says B.hi!'
```

Trace — `this = c` at *every* level; only the executing body changes:

| Step | Call | Resolves how | Body run | `this` |
|---|---|---|---|---|
| 1 | `c.greet()` | dynamic walk from `c`: `C.prototype` has `greet` | `C.greet` | `c` |
| 2 | `super.greet()` in `C.greet` | static: from `C.prototype.[[Prototype]]` = `B.prototype` (no `greet`) → `A.prototype` (found) | `A.greet` | `c` (preserved) |
| 3 | `this.hi()` in `A.greet` | dynamic walk from `c`: `C.prototype` (no `hi`) → `B.prototype` (found) | `B.hi` | `c` |

Result: `'C says ' + ('B.hi' + '!')` = `'C says B.hi!'`. The body migrated `C.greet → A.greet → B.hi`; `this` never rebound off `c`. Step 2 is static-start/dynamic-`this`; steps 1 and 3 are fully dynamic from the receiver.
