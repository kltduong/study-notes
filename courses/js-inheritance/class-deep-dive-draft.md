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

## 1. Unified dispatch rule

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

A method can be called in two forms:

- **Normal call:** `expr.method(args)` — e.g. `d.speak()`
- **Super call:** `super.method(args)` — e.g. `super.speak()` inside a method body

Both forms resolve **two independent questions**:

1. **Which function body runs?** — determined by a prototype-chain walk.
2. **What is `this` inside it?** — always the receiver (the object left of the dot).

These two questions are fully orthogonal — knowing one tells you nothing about the other. The only thing that differs between the two call forms is **where the chain walk starts** for question 1. Question 2's answer is always the same regardless of call form: the receiver, unchanged.

#### Question 1: which function body runs?

Both call forms find the function the same way — walk a `[[Prototype]]` chain until the property is found. They differ only in **where the walk begins**:

| Call form | Walk starts at | Example |
|---|---|---|
| `expr.method()` | `expr` itself (the receiver) | `d.speak()` → starts at `d`, walks up |
| `super.method()` | one level above where the calling method is written | inside `Dog.prototype.speak`: starts at `Animal.prototype` |

**Normal dispatch** is receiver-based — already derived in [`prototype-deep-dive.md`](prototype-deep-dive.md#custom-prototypes-and-this-binding); not re-derived here.

**Super dispatch** shifts the starting point. `super` is not an object — it's a lookup instruction meaning "skip my level." Concretely: start the walk at `Object.getPrototypeOf(HomeObject)`, where `HomeObject` is the object the calling method was *defined on*. This start point is fixed at definition time (static).

```
super.X  ≈  Object.getPrototypeOf(HomeObject)[X]
                        ▲
                 "one above where I'm written"
                 fixed at definition time
```

#### Question 2: what is `this`?

Always the receiver. `super` does not change it — the parent method runs with the same `this` the caller had.

```js
d.speak()          // this = d
  → super.speak() // this = still d (Animal.prototype.speak sees d)
```

Putting both together:

```
super.X(args)  ≈  Object.getPrototypeOf(HomeObject)[X].call(this, ...args)
                              ▲                                 ▲
                       Q1: where to find            Q2: receiver,
                       the function (static)        unchanged (dynamic)
```

#### `[[HomeObject]]` — what anchors `super`

`[[HomeObject]]` is an internal slot on the **function object itself**, set once when the method is defined with method syntax in a class or object-literal body. It records *which object this method was written into* — that's the reference point `super` uses to know where "my level" is.

It travels with the function. Detaching the method (`const f = c.greet`) breaks `this` (no longer called on a receiver) but not `super`'s start point (still anchored to the original HomeObject). The two fail independently because they're sourced from different times: `this` from call time, `super`'s start point from definition time.

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
