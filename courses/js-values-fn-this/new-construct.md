# 1. Constructor Calls (`new`)

**TL;DR:** `new` invokes a function's `[[Construct]]` internal method — a completely separate code path from `[[Call]]`. It creates a fresh object (via OrdinaryCreateFromConstructor, with `[[Prototype]]` from `Constructor.prototype`), binds it as `this`, sets `new.target`, runs the body, then applies the return-value override rule: if the body returns an Object type, that replaces the fresh object; primitives (including `null`) are ignored and the fresh object is used. Functions are constructable only if `[[IsConstructor]] = true` (set at creation time by syntactic form).

## 1.1. `[[Construct]]` vs `[[Call]]` — two internal methods

Every function object can have up to two internal methods for invocation:

| Internal method | Triggered by | Purpose |
|---|---|---|
| `[[Call]](thisValue, args)` | `f()`, `f.call()`, `f.apply()` | Run the function body with a supplied `this` |
| `[[Construct]](args, newTarget)` | `new f()`, `super()` | Create an object, run the body, return the object |

These are **separate code paths** — not "the same call with a flag." They share the function body, but everything around it differs:

| Step | `[[Call]]` | `[[Construct]]` |
|---|---|---|
| Receives `thisValue`? | Yes — from call site | No — creates its own |
| Creates a new object? | No | Yes — OrdinaryCreateFromConstructor |
| Sets `this` to... | The received `thisValue` | The freshly created object |
| Sets `new.target`? | No (`undefined`) | Yes — the constructor being `new`'d |
| Return-value override? | No — returns whatever the body returns | Yes — special rule applies |

Without `new`, a constructor is just a regular function — `this` follows the normal Reference-base rule (`undefined` in strict mode), and no object is created.

## 1.2. The `[[Construct]]` protocol

When `new Dog("Rex")` executes, the engine runs `Dog.[[Construct]](["Rex"], Dog)`:

### 1.2.1. Step 1: OrdinaryCreateFromConstructor

```
OrdinaryCreateFromConstructor(constructor, "%Object.prototype%"):
    1. proto = Get(constructor, "prototype")
       (falls back to %Object.prototype% if not an object)
    2. obj = OrdinaryObjectCreate(proto)
    3. return obj
```

Creates a **fresh ordinary object** whose `[[Prototype]]` is `Constructor.prototype`. The object exists before the function body runs.

Key detail: the engine reads `.prototype` at construction time. Reassigning `.prototype` later doesn't affect already-created instances.

### 1.2.2. Step 2: Bind `this` = the fresh object

The fresh object becomes the `thisValue` passed to OrdinaryCallBindThis → written into the ER's `[[ThisValue]]` slot. Now `this` inside the body refers to the fresh object.

### 1.2.3. Step 3: Set `new.target`

`newTarget` (the constructor that `new` was applied to) is stored in the EC. Accessible as `new.target` inside the body.

### 1.2.4. Step 4: Run the body

Parameters bound normally. Body executes with `this` = fresh object, `new.target` = constructor.

### 1.2.5. Step 5: Return-value override

```
Let result = body's completion value

if Type(result) is Object:
    return result           ← the explicit return wins
else:
    return thisValue        ← the fresh object wins
```

## 1.3. Return-value override

### 1.3.1. The type gate

| Value | `Type(...)` | Override fires? |
|---|---|---|
| `{}`, `[]`, `function(){}`, `new Map()` | Object | Yes — replaces fresh object |
| `null` | Null | No — fresh object used |
| `undefined` (no return) | Undefined | No — fresh object used |
| `42`, `"str"`, `true`, `Symbol()`, `0n` | primitive types | No — fresh object used |

`null` is the surprising one — despite `typeof null === "object"` (the famous bug), the spec's `Type(null)` is Null, not Object. So `return null` from a constructor is ignored.

### 1.3.2. The common case: no explicit return

```js
function Cat(name) {                                  // L1
  this.name = name;                                   // L2
}                                                     // L3

const c = new Cat("Whiskers");                        // L4
// completion value = undefined → not Object → fresh object returned
// c = { name: "Whiskers" }, [[Prototype]] = Cat.prototype
c instanceof Cat;                                     // L5 → true
```

### 1.3.3. When the override breaks `instanceof`

```js
function Dog(name) {                                  // L1
  this.name = name;                                   // L2
  return { breed: "unknown" };                        // L3
}                                                     // L4

const d = new Dog("Rex");                             // L5
// Type({ breed: "unknown" }) is Object → override fires
// d = { breed: "unknown" } — a plain object literal
// The fresh object (with [[Prototype]] = Dog.prototype) is discarded
d instanceof Dog;                                     // L6 → false
```

The fresh object that *had* `Dog.prototype` in its chain was thrown away. `d` is a plain object with `[[Prototype]] = Object.prototype`.

## 1.4. `new.target`

### 1.4.1. What it is

`new.target` is a **meta-property** — a single syntactic unit (not the `new` operator applied to `target`). The parser treats it as one token, like `import.meta`.

| Call mode | `new.target` value |
|---|---|
| `new Foo()` | `Foo` (the constructor function) |
| `Foo()` (plain call) | `undefined` |
| `super()` in derived class | The derived constructor (forwarded) |
| Arrow function | Inherited from enclosing scope (chain walk) |

### 1.4.2. How it's set

- `[[Construct]]`: stores `newTarget` in the EC's `[[NewTarget]]` field.
- `[[Call]]`: `[[NewTarget]]` = `undefined`.

### 1.4.3. Primary use: enforcing `new`

```js
"use strict";                                         // L1
function Person(name) {                               // L2
  if (!new.target) {                                  // L3
    throw new TypeError("Must use new");              // L4
  }                                                   // L5
  this.name = name;                                   // L6
}                                                     // L7

new Person("Alice");                                  // L8 → works
Person("Bob");                                        // L9 → TypeError
```

> **Aside —** Before `new.target` (ES6), the common hack was `this instanceof Foo` — unreliable because `Foo.call(existingInstance)` makes it `true` without `new`. `new.target` is the correct, unforgeable check.

### 1.4.4. Self-healing constructor pattern

```js
"use strict";                                         // L1
function Factory(name) {                              // L2
  if (!new.target) {                                  // L3
    return new Factory(name);                         // L4 — redirect to [[Construct]]
  }                                                   // L5
  this.name = name;                                   // L6
}                                                     // L7

Factory("X");                                         // L8 → works (redirects internally)
new Factory("Y");                                     // L9 → works (direct)
```

Both produce a Factory instance. The plain call at L8 detects `!new.target`, calls `new Factory(name)` internally, and returns the result.

### 1.4.5. `new.target` in arrows

Same pattern as `this` and `arguments` — arrows don't have their own `new.target`. Resolution walks `[[OuterEnv]]` to the enclosing function's EC.

## 1.5. `[[IsConstructor]]` — the structural gate

Before `[[Construct]]` runs, the engine checks `[[IsConstructor]]`. If `false` → **TypeError** before any construction logic executes.

### 1.5.1. Which functions are constructable

Set at function creation time by syntactic form:

| Function kind | `[[IsConstructor]]` | Rationale |
|---|---|---|
| Function declaration | `true` | Legacy — pre-ES6 all functions were constructable |
| Function expression | `true` | Same legacy reason |
| Arrow function | `false` | No `this` machinery → can't receive fresh object |
| Method shorthand (`{ foo() {} }`) | `false` | Methods aren't standalone constructors |
| Async function | `false` | Returns a Promise, not the fresh object |
| Generator function | `false` | Returns an iterator, not the fresh object |
| Async generator | `false` | Same — returns async iterator |
| `class` constructor | `true` | Explicitly designed for `new` |

**Modern defaults:** Use method shorthand for object/class methods (not constructable, has `super`). Use `class` for constructors. Use arrow functions for inline callbacks. The function-expression-as-property form (`key: function() {}`) is legacy — accidentally constructable, no `super`, wastes memory on a `.prototype` object. No reason to use it in new code.

### 1.5.2. The `class` asymmetry

Class constructors have `[[IsConstructor]] = true` but also **require** `new`:

```js
class Foo { constructor() { this.x = 1; } }          // L1
Foo();                                                // L2 → TypeError: cannot invoke without 'new'
```

The class `[[Call]]` implementation explicitly checks and throws — it's not that `[[Call]]` is absent, it's that it rejects. Inverse of arrows: arrows can call but can't `new`; classes can `new` but can't call.

## 1.6. Interaction with `bind`

### 1.6.1. `[[Construct]]` ignores `[[BoundThis]]`

When you `new` a bound function, the BoundFunction's `[[Construct]]` method:

1. Prepends `[[BoundArguments]]` to call-time args.
2. **Ignores `[[BoundThis]]`** — construction creates a fresh object, not reuses an existing one.
3. Substitutes `newTarget`: if `newTarget === boundFn`, sets `newTarget = [[BoundTargetFunction]]`.
4. Delegates to `[[BoundTargetFunction]].[[Construct]](args, newTarget)`.

The `newTarget` substitution (step 3) ensures the fresh object's `[[Prototype]]` comes from the **original function's** `.prototype`, not the bound function (which has no `.prototype`).

```js
"use strict";                                         // L1
function Dog(name) {                                  // L2
  this.name = name;                                   // L3
}                                                     // L4

const BoundDog = Dog.bind({ x: "ignored" });          // L5
const d = new BoundDog("Rex");                        // L6

console.log(d.name);                                  // L7 → "Rex"
console.log(d instanceof Dog);                        // L8 → true
console.log(d.x);                                     // L9 → undefined
```

**Trace of L6:**
1. `new BoundDog("Rex")` → `BoundDog.[[Construct]](["Rex"], BoundDog)`
2. BoundFunction `[[Construct]]`:
   - `newTarget === BoundDog === F` → set `newTarget = Dog` (the `[[BoundTargetFunction]]`)
   - args = concat(`[[BoundArguments]]` = [], ["Rex"]) = ["Rex"]
   - `[[BoundThis]]` = `{ x: "ignored" }` → **not used** (`[[Construct]]` ignores it)
   - Delegates: `Dog.[[Construct]](["Rex"], Dog)`
3. Dog's `[[Construct]]`:
   - OrdinaryCreateFromConstructor(`Dog`, "%Object.prototype%") → `Get(Dog, "prototype")` = `Dog.prototype` → fresh object with `[[Prototype]] = Dog.prototype`
   - `this` = fresh object, body runs, `this.name = "Rex"`
   - No explicit return → fresh object returned
4. `d` = `{ name: "Rex" }` with `[[Prototype]] = Dog.prototype`
5. `d instanceof Dog` → `true` ✓

### 1.6.2. `bind` for partial application with `new`

The useful part of `bind` under `new` is argument pre-filling:

```js
"use strict";                                         // L1
function Point(x, y) {                                // L2
  this.x = x;                                        // L3
  this.y = y;                                        // L4
}                                                     // L5

const PointOnXAxis = Point.bind(null, 0);             // L6
const p = new PointOnXAxis(5);                        // L7
// p = { x: 0, y: 5 }, [[Prototype]] = Point.prototype
```

### 1.6.3. Summary: `bind` under `[[Call]]` vs `[[Construct]]`

| Aspect | `boundFn()` (`[[Call]]`) | `new boundFn()` (`[[Construct]]`) |
|---|---|---|
| `[[BoundThis]]` | Used — replaces `thisValue` | Ignored — fresh object is `this` |
| `[[BoundArguments]]` | Prepended to args | Prepended to args |
| `newTarget` | N/A | Substituted to `[[BoundTargetFunction]]` |
| Prototype of result | N/A | From `[[BoundTargetFunction]].prototype` |

## 1.7. Quick reference

- **Two internal methods** — `[[Call]]` and `[[Construct]]` are separate code paths. `new` triggers `[[Construct]]`; plain calls trigger `[[Call]]`.
- **`[[Construct]]` protocol** — OrdinaryCreateFromConstructor (fresh object with `[[Prototype]] = F.prototype`) → bind `this` = fresh object → set `new.target` → run body → return-value override.
- **Return-value override** — explicit return of an Object type replaces the fresh object. Primitives and `null` are ignored; fresh object is used.
- **`new.target`** — meta-property; equals the constructor under `new`, `undefined` under plain call. Unforgeable `new`-detection.
- **`[[IsConstructor]]`** — structural gate checked before `[[Construct]]` runs. Arrows, methods, async, generators are `false`. Function declarations/expressions are `true` (legacy). Classes are `true` (by design).
- **`bind` + `new`** — `[[BoundThis]]` ignored, `[[BoundArguments]]` prepended, `newTarget` substituted to original function.
