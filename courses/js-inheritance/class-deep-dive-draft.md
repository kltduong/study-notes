# 1. Class Deep Dive — Draft

## 1.1. Plan (teaching order)

- [x] 1. **Unified dispatch rule** — `this` from receiver (recap + forward link); `super.method()` as lookup-redirect with `this` preserved. Worked synthesis: A/B/C cascade.
- [x] 2. **Polymorphism payoff** — inherited body's `this.method()` resolves to most-derived override because `this` never rebinds.
- [x] 3. **`super(...)` in derived constructors** — full uninitialized-`this` mechanics, unconditional ReferenceError.
- [x] 4. **Static dispatch + constructor-side chain** — `this` inside static = class on left of dot; `super.staticMethod()` on constructor chain.
- [ ] 5. **Class fields vs prototype methods (inheritance lens)** — brief recap; inheritance implications.
- [ ] 6. **Private fields (`#field`)** — class-only syntax, spec-enforced encapsulation.
- [ ] 7. **Where "just sugar" leaks** — no-`new` TypeError, TDZ on class binding, `new.target`.
- [ ] 8. **Extending built-ins** — `extends Array/Error/Map`, `Symbol.species`.

---

## 1.2. Unified dispatch rule

### 1.2.1. Teaser

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

### 1.2.2. The axiom

A method call resolves **two independent questions**:

1. **Which function body runs?** — determined by a prototype-chain walk.
2. **What is `this` inside that body?** — always the receiver (the object left of the dot at the original call site).

#### 1.2.2.1. Question 1: which function body runs?

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

#### 1.2.2.2. Question 2: what is `this`?

Always the receiver. `super` does not rebind `this` — the parent method runs with the same `this` the caller had. This is what makes inheritance useful: a parent method can reference `this.prop` and get the child instance's data.

```js
d.speak()          // this = d
  → super.speak() // this = still d (Animal.prototype.speak sees d)
```

#### 1.2.2.3. The full desugaring

Putting Q1 and Q2 together into one pseudo-expression:

```
super.X(args)  ≈  Object.getPrototypeOf(HomeObject)[X].call(this, ...args)
                              ▲                                 ▲
                       Q1: where to find the             Q2: receiver,
                       function body (static)            unchanged (dynamic)
```

Two inputs, two different lifetimes: the lookup target is frozen at definition time; `this` flows in at call time. This separation is the entire mechanism — everything else (polymorphism, constructor chaining, the proof below) falls out of it.

#### 1.2.2.4. `[[HomeObject]]` — what anchors `super`

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

##### 1.2.2.4.1. Demo — detach breaks `this`, not `super`

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

### 1.2.3. Why it *must* be HomeObject-based (not receiver-based)

Proof by contradiction. Suppose `super.X` started its lookup from the parent of **`this`'s class** (receiver-based):

```js
class A { greet() { return 'A→' + (super.greet?.() ?? 'end'); } }  // L1
class B extends A {}                                                // L2 — no greet
class C extends B {}                                                // L3 — no greet
new C().greet();                                                    // A.greet runs, this = c
```

Inside `A.greet`, `this` is `c` (class `C`, parent `B`). Receiver-based reading: look up `greet` from `B.prototype` → not found → `A.prototype` → **finds `A.greet` again** → calls it → `super.greet` again → `B.prototype` → `A.prototype` → `A.greet` → **infinite recursion**.

Calling `super` from an inherited method is legal, everyday code. A rule that makes it infinitely recurse cannot be the rule. Therefore `super`'s start point must be **static** (`HomeObject.[[Prototype]]`), not receiver-derived. Under the real rule: `A.greet`'s `[[HomeObject]]` = `A.prototype`, so `super.greet` starts at `A.prototype.[[Prototype]]` = `Object.prototype` → no `greet` → terminates. No loop. The static rule is the *only* coherent choice.

### 1.2.4. Worked synthesis — the A/B/C cascade

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

## 1.3. Polymorphism payoff

Polymorphism is not a separate feature. It is **Q1 ∘ Q2** — the composition of the two coordinates — applied to a `this.method()` call inside an inherited method.

```js
class Animal {
  speak() { return `${this.sound()} (from Animal.speak)`; }  // defined only on Animal
  sound() { return 'generic noise'; }
}
class Dog extends Animal {
  sound() { return 'woof'; }                                  // override
}

new Dog().speak();   // 'woof (from Animal.speak)'
```

`speak` exists only on `Animal.prototype`. Trace `new Dog().speak()`:

1. `d.speak()` — dynamic walk from `d`: `Dog.prototype` (no `speak`) → `Animal.prototype` (found). Body = `Animal.speak`, **`this = d`**.
2. Inside `Animal.speak`, `this.sound()` — this is an *ordinary receiver-based call*. Q2 kept `this = d`, so Q1 now runs a **fresh dynamic walk from `d`**: `Dog.prototype.sound` (found, step 0) before `Animal.prototype.sound`. Body = `Dog.sound`.

Result: `'woof (from Animal.speak)'`. The parent method called *down into* the child override without naming — or even knowing about — the child. That downward reach is the entire payoff:

> A method written in the parent, calling `this.x()`, automatically dispatches to whatever override the actual receiver provides — because `this` is preserved (Q2) and `this.x()` re-dispatches dynamically from it (Q1).

This is what makes inheritance *useful* rather than merely *structural*. Without `this`-preservation, a parent method's `this.sound()` would resolve against the parent and every subclass would have to re-implement `speak` just to call its own `sound`. Polymorphism falls out of the axiom for free — no extra mechanism.

> **Aside —** the "template method pattern" (a fixed parent algorithm with overridable steps) is just this, named. `Animal.speak` is the template; `sound` is the overridable hole.

The flip side — if `sound` were a **class field** (`sound = () => 'woof'`) rather than a method, polymorphism still works but via a *different* path: the field is an own property on `d`, found at walk step 0 directly, never reaching any prototype. Same observable result, different mechanism. That asymmetry is sub-part 5.

## 1.4. `super(...)` in derived constructors

Touched in [`building-chains.md`](building-chains.md) as the constructor-side counterpart to `Animal.call(this, name)`. The full mechanic: **`super()`'s job is to bring the instance into existence and bind `this` to it** — not merely to "let you use `this`."

### 1.4.1. The two-failure model

A base (non-derived) constructor: `new` allocates the instance *before* the body runs, so `this` is bound from line 1.

A derived constructor: the instance is allocated by running *up* the `super()` chain. Until `super()` returns, the `this` *binding* is **uninitialized** — a TDZ-like state for the `this` binding specifically (not `this === undefined`; the binding itself is unusable). This produces **two** distinct ReferenceErrors:

| Failure | Trigger | Why |
|---|---|---|
| Early use | read *or write* `this` before `super()` returns | binding not yet initialized |
| Missing instance | derived constructor finishes without `super()` having run | no instance was ever created → nothing to return |

```js
class Base { constructor() { this.tag = 'base'; } }

class Derived extends Base {
  constructor() { this.x = 1; super(); }   // ReferenceError at `this.x` — early use
}

class D extends Base {
  constructor() { return; }                // ReferenceError — finished, no super(), no instance
}                                           // (body never even mentions `this`)

class Plain { constructor() { this.x = 1; } }   // fine — base ctor, `new` pre-made the instance
```

### 1.4.2. Why the rule is unconditional

The engine is **not** analyzing "does this `this` use depend on parent setup?" Before `super()` there is *no instance in existence at all* — so every path that needs a bound `this` fails identically: `this.x = 1`, `console.log(this)`, `const y = this`, and even an empty `return;` (which implicitly yields `this`). `super()` is the conduit: it runs the parent constructor, which (at the base-most level) allocates the object using **`new.target.prototype`** for the `[[Prototype]]` link — so `new Derived()` still produces a `Derived` instance even though allocation physically happens inside `Base`. Allocated at the bottom of the chain, shaped by `new.target` at the top, threaded back down through each `super()`.

> **Aside —** escape hatch: a derived constructor *may* skip `super()` if it explicitly `return`s its own object (`return { custom: true }`). An explicit object return value means there's no missing-instance error. Niche — listed only so the model is complete: the invariant is "a derived constructor must yield an object," normally satisfied by `super()` creating `this`.

### 1.4.3. Resolving the building-chains refinement

The earlier refinement modeled this as conditional ("fails only when the `this` use depends on parent setup"). It is unconditional, and now the *why* is structural: `super()` doesn't grant permission to touch `this` — it **creates the thing `this` names**. No creation → nothing to touch, nothing to return. Conditionality never enters.

## 1.5. Static dispatch + the constructor-side chain

The sub-part 1 axiom **does not change** for static methods. Same two coordinates (which-function / what-`this`), same `super` = lookup-redirect. The *only* substitution: the relevant chain is the **constructor-side** chain, and `[[HomeObject]]` for a static method is the **constructor function itself**, not `.prototype`.

Recall from [`building-chains.md`](building-chains.md) that `class Dog extends Animal` wires *two* chains:

- Instance side: `Dog.prototype.[[Prototype]] = Animal.prototype` — for `dog.method()`.
- Constructor side: `Dog.[[Prototype]] = Animal` — for `Dog.staticMethod()`.

### 1.5.1. `this` in a static method = the class it was called on

A static method is just a property on the constructor function object. Calling it is `Constructor.method()` — ordinary receiver-based dispatch (Q2), where the "receiver" is the constructor left of the dot.

```js
class Animal {
  static create(name) { return new this(name); }   // this = the class called on
  constructor(name) { this.name = name; }
}
class Dog extends Animal {
  constructor(name) { super(name); this.kind = 'dog'; }
}

Animal.create('x').constructor.name;   // 'Animal' — this = Animal, new Animal(...)
Dog.create('y').constructor.name;      // 'Dog'    — this = Dog,    new Dog(...)
```

`create` is defined only on `Animal`. `Dog.create('y')` resolves it via the **constructor-side** chain (`Dog.[[Prototype]] = Animal`), and `this` is `Dog` (left of the dot). `new this(name)` therefore constructs a `Dog`. This is polymorphism (sub-part 2) on the constructor chain: the inherited static method "constructs down into" the subclass because `this` is the receiver, unchanged. (Already demoed in `building-chains.md`; the *mechanism* is identical to instance-side polymorphism — only the chain differs.)

### 1.5.2. `super.staticMethod()` — same redirect, constructor `[[HomeObject]]`

Inside a **static** method, `super` refers to the parent **constructor**, not the parent prototype. The lookup-redirect rule from sub-part 1 is unchanged; only the `[[HomeObject]]` substitution differs:

| Method kind | `[[HomeObject]]` | `super.X` looks up from |
|---|---|---|
| Instance method | `C.prototype` | `Object.getPrototypeOf(C.prototype)` = parent's `.prototype` |
| **Static method** | `C` (the constructor fn) | `Object.getPrototypeOf(C)` = **parent constructor** |

```js
class Animal {
  static describe() { return 'an animal'; }
}
class Dog extends Animal {
  static describe() { return super.describe() + ' that barks'; }  // [[HomeObject]] = Dog
}

Dog.describe();   // 'an animal that barks'
```

Trace: `Dog.describe()` → dynamic walk from `Dog` finds `Dog.describe`, `this = Dog`. `super.describe()` → static start from `Object.getPrototypeOf(Dog)` = `Animal` → `Animal.describe` found, `this` preserved (`Dog`). Identical shape to the A/B/C cascade in sub-part 1 — substitute "constructor" for "prototype" and nothing else moves.

The unifying statement for the whole chunk so far: **one dispatch rule, two chains.** Instance methods walk the `.prototype` chain; static methods walk the constructor chain; `super` is a definition-time lookup-redirect on whichever chain the method's `[[HomeObject]]` sits on; `this` is always the call-time receiver, preserved across `super`.
