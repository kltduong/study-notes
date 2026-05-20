# 1. Constructor Calls (`new`) — Teaching Draft

## 1.1. Plan (teaching order)

- [x] `[[Construct]]` vs `[[Call]]` — two internal methods, one function object. What `new` actually invokes.
- [x] The `[[Construct]]` protocol — OrdinaryCreateFromConstructor, `this` = fresh object, body runs, return-value override rule.
- [x] Return-value override — the Object-type gate, why primitives are ignored, what "no return" means.
- [x] `new.target` — what it is, how it's set, what it enables (detecting `new` vs plain call).
- [x] `[[IsConstructor]]` — which functions have it, what happens without it, the structural gate.
- [x] Interaction with `bind` — `[[Construct]]` on BoundFunction, `[[BoundThis]]` ignored, `new.target` passthrough.

---

## 1.2. `[[Construct]]` vs `[[Call]]` — two internal methods

### 1.2.1. The fundamental split

Every function object can have up to two internal methods for invocation:

| Internal method | Triggered by | Purpose |
|---|---|---|
| `[[Call]](thisValue, args)` | `f()`, `f.call()`, `f.apply()` | Run the function body with a supplied `this` |
| `[[Construct]](args, newTarget)` | `new f()`, `super()` | Create an object, run the body, return the object |

These are **separate code paths** — not "the same call with a flag." They share the function body, but everything around it differs:

| Step | `[[Call]]` | `[[Construct]]` |
|---|---|---|
| Receives `thisValue`? | Yes — from call site (Reference base, `call`/`apply`, `bind`) | No — creates its own |
| Creates a new object? | No | Yes — OrdinaryCreateFromConstructor |
| Sets `this` to... | The received `thisValue` | The freshly created object |
| Sets `new.target`? | No (`undefined`) | Yes — the constructor being `new`'d |
| Return-value override? | No — returns whatever the body returns | Yes — special rule applies |

### 1.2.2. Teaser reveal — why L7 throws

```js
"use strict";                                         // L1
function Dog(name) {                                  // L2
  this.name = name;                                   // L3
  return { breed: "unknown" };                        // L4
}                                                     // L5

const d2 = Dog("Rex");                                // L7
```

L7 is `Dog("Rex")` — no `new`. The engine invokes `Dog.[[Call]](thisValue, ["Rex"])`.

What's `thisValue`? The call expression `Dog("Rex")` produces a Reference with base = the script's ER (it's a plain identifier, not a member expression). ER base → `thisValue = undefined` (strict mode).

So `Dog.[[Call]](undefined, ["Rex"])` runs:
1. OrdinaryCallBindThis writes `undefined` into Dog's ER `[[ThisValue]]` slot.
2. Body executes. L3: `this.name = name` → `undefined.name = "Rex"` → **TypeError**.

No object was ever created. `[[Call]]` doesn't create objects — that's `[[Construct]]`'s job.

### 1.2.3. The mental model

Think of `new` as invoking a **different entry point** on the same function object. The function body is shared code, but the setup and teardown around it are completely different protocols.

```
┌─────────────────────────────────────────┐
│           Function Object (Dog)         │
├──────────────────┬──────────────────────┤
│   [[Call]]       │   [[Construct]]      │
│                  │                      │
│ 1. Receive this  │ 1. Create new obj    │
│ 2. Bind this     │ 2. Bind this = obj   │
│ 3. Run body      │ 3. Set new.target    │
│ 4. Return result │ 4. Run body          │
│                  │ 5. Return override   │
└──────────────────┴──────────────────────┘
```

---

## 1.3. The `[[Construct]]` protocol — step by step

When `new Dog("Rex")` executes, the engine runs `Dog.[[Construct]](["Rex"], Dog)`. Here's the full sequence:

### 1.3.1. Step 1: OrdinaryCreateFromConstructor

**OrdinaryCreateFromConstructor** is a spec-level abstract operation — the mechanism that answers "where does the new object come from and what's its prototype chain?" It runs *before* the constructor body executes, producing the object that `this` will point to inside the body.

The operation takes two arguments: the constructor function and a fallback intrinsic prototype (used if the constructor's `.prototype` isn't a valid object).

```
OrdinaryCreateFromConstructor(Dog, "%Object.prototype%"):
    1. proto = Get(Dog, "prototype")
       → if Type(proto) is not Object, use the fallback (%Object.prototype%)
    2. obj = OrdinaryObjectCreate(proto)
       → allocate a fresh ordinary object with [[Prototype]] = proto
    3. return obj
```

What this produces: a **fresh ordinary object** whose `[[Prototype]]` is `Dog.prototype`. The object exists before the function body runs — it's the "blank canvas" that the constructor will populate with properties.

Why it matters:
- It's the reason `new Dog()` instances have access to methods on `Dog.prototype` — the chain is wired here, not by the constructor body.
- The constructor body's job is just to add own-properties to this already-linked object.
- The return-value override (Step 5) can discard this object entirely — but OrdinaryCreateFromConstructor always runs regardless.

Key detail: the engine reads `Dog.prototype` at construction time. If you reassign `Dog.prototype` later, already-created instances keep their original `[[Prototype]]`. New instances get the new one.

### 1.3.2. Step 2: Bind `this` = the fresh object

The fresh object becomes the `thisValue` passed to OrdinaryCallBindThis. Since `Dog` is an ordinary function (not an arrow), OrdinaryCallBindThis writes it into the ER's `[[ThisValue]]` slot.

Now `this` inside the constructor body refers to the fresh object.

### 1.3.3. Step 3: Set `new.target`

The second argument to `[[Construct]]` is `newTarget` — the constructor that `new` was applied to. For a direct `new Dog()`, `newTarget = Dog`. (It differs from the running function only with `super()` calls in derived classes — covered in the class chunk.)

`new.target` is stored in the EC and accessible inside the function body.

### 1.3.4. Step 4: Run the body

The function body executes with:
- `this` = the fresh object
- `new.target` = the constructor
- Parameters bound normally

```js
function Dog(name) {                                  // L1
  // this = {} (fresh object, [[Prototype]] = Dog.prototype)
  this.name = name;                                   // L2 — writes 'name' property on the fresh object
  return { breed: "unknown" };                        // L3
}                                                     // L4
```

### 1.3.5. Step 5: Return-value override rule

After the body completes, `[[Construct]]` applies a special rule to the return value:

```
Let result = body's completion value

if Type(result) is Object:
    return result           ← the explicit return wins
else:
    return thisValue        ← the fresh object wins (ignores the return)
```

This is the **return-value override**: if the constructor explicitly returns an Object (capital-O — anything that's not a primitive), that object replaces the auto-created one. If the constructor returns a primitive (or has no `return` statement, which is `undefined`), the fresh object is used.

---

## 1.4. Return-value override — the full picture

### 1.4.1. Traced through the teaser

```js
const d1 = new Dog("Rex");                            // L6
```

1. `Dog.[[Construct]](["Rex"], Dog)` invoked.
2. OrdinaryCreateFromConstructor → `obj = {}` with `[[Prototype]] = Dog.prototype`.
3. OrdinaryCallBindThis → ER `[[ThisValue]] = obj`.
4. `new.target = Dog`.
5. Body runs:
   - L3: `this.name = "Rex"` → `obj.name = "Rex"`. The fresh object now has a `name` property.
   - L4: `return { breed: "unknown" }` → body returns a new object literal.
6. Return-value override: `Type({ breed: "unknown" })` is Object → **use the returned object, discard `obj`**.
7. `d1 = { breed: "unknown" }`.

The fresh object (with `name: "Rex"` and `[[Prototype]] = Dog.prototype`) is created, populated, and then **thrown away** because the explicit return is an Object.

### 1.4.2. Why `d1 instanceof Dog` is `false`

`instanceof` checks whether `Dog.prototype` exists in `d1`'s prototype chain. But `d1` is `{ breed: "unknown" }` — a plain object literal whose `[[Prototype]]` is `Object.prototype`. The fresh object that *had* `Dog.prototype` in its chain was discarded by the override rule.

### 1.4.3. The common case: no explicit return

```js
function Cat(name) {                                  // L1
  this.name = name;                                   // L2
  // no return statement → completion value is undefined
}                                                     // L3

const c = new Cat("Whiskers");                        // L4
// Type(undefined) is not Object → use the fresh object
// c = { name: "Whiskers" }, [[Prototype]] = Cat.prototype
c instanceof Cat;                                     // L5 → true
```

Most constructors don't return anything. The override rule's `else` branch fires, and the fresh object (now populated by the body) is the result. This is the "normal" constructor behavior.

### 1.4.4. Primitive returns are ignored

```js
function Weird() {                                    // L1
  this.x = 1;                                        // L2
  return 42;                                          // L3 — primitive, ignored by [[Construct]]
}                                                     // L4

const w = new Weird();                                // L5
console.log(w);                                       // L6 → { x: 1 }
console.log(w instanceof Weird);                      // L7 → true
```

`42` is a Number primitive. `Type(42)` is not Object → override rule ignores it → fresh object returned. The `return 42` is effectively dead code under `new`.

### 1.4.5. The type gate — what counts as "Object"

The override checks `Type(result)` — the spec's type classification:

| Value | `Type(...)` | Override fires? |
|---|---|---|
| `{}`, `[]`, `function(){}`, `new Map()` | Object | Yes — replaces fresh object |
| `null` | Null | No — fresh object used |
| `undefined` (no return) | Undefined | No — fresh object used |
| `42`, `"str"`, `true`, `Symbol()`, `0n` | primitive types | No — fresh object used |

`null` is the surprising one — despite `typeof null === "object"` (the famous bug), the spec's `Type(null)` is Null, not Object. So `return null` from a constructor is ignored, and the fresh object is used.

---

## 1.5. `new.target` — detecting the call mode

### 1.5.1. What it is

`new.target` is a **meta-property** (like `import.meta`) — not a property access on an object called `new`. It's a single token that the engine resolves from the current execution context.

| Call mode | `new.target` value |
|---|---|
| `new Foo()` | `Foo` (the constructor function) |
| `Foo()` (plain call) | `undefined` |
| `super()` in derived class | The derived constructor (the one `new` was applied to) |
| Arrow function | Inherited from enclosing scope (same chain-walk as `this`) |

### 1.5.2. How it's set

During `[[Construct]](args, newTarget)`:
1. The engine creates the new EC for the function.
2. It stores `newTarget` in the EC's `[[NewTarget]]` field.
3. When code reads `new.target`, the engine returns `EC.[[NewTarget]]`.

During `[[Call]]`:
1. `[[NewTarget]]` is set to `undefined`.
2. Reading `new.target` returns `undefined`.

### 1.5.3. The primary use case: enforcing `new`

Before `class` syntax (which enforces `new` automatically), constructors were plain functions — callable both ways. `new.target` lets you detect and reject plain calls:

```js
"use strict";                                         // L1
function Person(name) {                               // L2
  if (!new.target) {                                  // L3
    throw new TypeError("Must use new");              // L4
  }                                                   // L5
  this.name = name;                                   // L6
}                                                     // L7

new Person("Alice");                                  // L8 → works, new.target = Person
Person("Bob");                                        // L9 → TypeError: Must use new
```

### 1.5.4. The secondary use case: identifying the actual constructor

In inheritance chains (covered more in the class chunk), `new.target` tells you which constructor was *originally* `new`'d — not which function is currently executing:

```js
"use strict";                                         // L1
function Base() {                                     // L2
  console.log(new.target.name);                       // L3
}                                                     // L4

function Derived() {                                  // L5
  Base.call(this);                                    // L6 — plain call, new.target = undefined inside Base
}                                                     // L7

new Base();                                           // L8 → logs "Base"
new Derived();                                        // L9 → logs undefined (Base was [[Call]]'d, not [[Construct]]'d)
```

With `class` and `super()`, this works differently — `super()` forwards `new.target` from the derived constructor to the base. But that's the class chunk's territory.

### 1.5.5. `new.target` in arrows

Arrows don't have their own `new.target` — same pattern as `this` and `arguments`. The engine walks `[[OuterEnv]]` to find the enclosing function's `new.target`:

```js
"use strict";                                         // L1
function Outer() {                                    // L2
  const check = () => new.target;                     // L3
  return check();                                     // L4
}                                                     // L5

new Outer();                                          // L6 → Outer (arrow inherits from Outer's EC)
Outer();                                              // L7 → undefined
```

> **Aside —** `new.target` was added in ES6 alongside `class`. Before ES6, the common hack was `this instanceof Foo` — but that's unreliable (fails with `Foo.call(existingInstance)` which makes `this instanceof Foo` true without `new`). `new.target` is the correct, unforgeable check.

---

## 1.6. `[[IsConstructor]]` — the structural gate

### 1.6.1. The check

Before `[[Construct]]` ever runs, the engine checks whether the function object has `[[IsConstructor]] = true`. If not → **TypeError: X is not a constructor**. This is a hard gate — no part of the `[[Construct]]` protocol executes.

### 1.6.2. Which functions are constructable

The rule is set at **function creation time** based on the function's syntactic form:

| Function kind | `[[IsConstructor]]` | Rationale |
|---|---|---|
| Function declaration | `true` | Legacy — all pre-ES6 functions were constructable |
| Function expression | `true` | Same as above |
| Arrow function | `false` | No `this` machinery → can't receive the fresh object |
| Method shorthand (`{ foo() {} }`) | `false` | Methods aren't standalone constructors |
| Async function | `false` | Returns a Promise, not the fresh object |
| Generator function | `false` | Returns an iterator, not the fresh object |
| Async generator | `false` | Same — returns async iterator |
| `class` constructor | `true` | Explicitly designed for `new` |

> **Aside —** Function declarations/expressions being constructable is a legacy artifact. Pre-ES6, there was no syntactic distinction between "function meant as constructor" and "function meant as regular function." Every function got `[[IsConstructor]] = true`. ES6 introduced new forms (arrows, methods, async, generators) that are explicitly *not* constructable — the language learned to distinguish intent.

**Modern defaults:** Use method shorthand for object/class methods (not constructable, has `super`). Use `class` for constructors. Use arrow functions for inline callbacks. The function-expression-as-property form (`key: function() {}`) is legacy — it's over-powered for a method (accidentally constructable, no `super`, wastes memory on a `.prototype` object). No reason to use it in new code.

### 1.6.3. The asymmetry with `class`

`class` constructors have `[[IsConstructor]] = true` but also a separate check: they **require** `new`. A class constructor called without `new` throws:

```js
class Foo {                                           // L1
  constructor() { this.x = 1; }                       // L2
}                                                     // L3

new Foo();                                            // L4 → works
Foo();                                                // L5 → TypeError: Class constructor Foo cannot be invoked without 'new'
```

This is the inverse of the arrow situation:
- Arrow: has `[[Call]]` but no `[[Construct]]` → can call, can't `new`
- Class constructor: has both, but `[[Call]]` throws → can `new`, can't call

The class `[[Call]]` implementation explicitly checks and throws — it's not that `[[Call]]` is absent, it's that it rejects.

### 1.6.4. Checking constructability at runtime

`Reflect.construct` and `new` both trigger the `[[IsConstructor]]` check. You can test it without actually constructing:

```js
function isConstructor(fn) {                          // L1
  try {                                               // L2
    Reflect.construct(String, [], fn);                 // L3 — uses fn as newTarget only
    return true;                                      // L4
  } catch {                                           // L5
    return false;                                     // L6
  }                                                   // L7
}                                                     // L8

isConstructor(function() {});                         // L9 → true
isConstructor(() => {});                              // L10 → false
isConstructor({ foo() {} }.foo);                      // L11 → false
```

> **Aside —** There's no direct `[[IsConstructor]]` accessor in the language. The `Reflect.construct` trick works because the spec checks `IsConstructor(newTarget)` — if `fn` isn't constructable, it throws before doing anything else.

