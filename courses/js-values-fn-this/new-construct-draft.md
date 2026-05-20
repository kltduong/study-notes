# Constructor Calls (`new`) — Teaching Draft

## Plan (teaching order)

- [x] `[[Construct]]` vs `[[Call]]` — two internal methods, one function object. What `new` actually invokes.
- [x] The `[[Construct]]` protocol — OrdinaryCreateFromConstructor, `this` = fresh object, body runs, return-value override rule.
- [x] Return-value override — the Object-type gate, why primitives are ignored, what "no return" means.
- [ ] `new.target` — what it is, how it's set, what it enables (detecting `new` vs plain call).
- [ ] `[[IsConstructor]]` — which functions have it, what happens without it, the structural gate.
- [ ] Interaction with `bind` — `[[Construct]]` on BoundFunction, `[[BoundThis]]` ignored, `new.target` passthrough.

---

## `[[Construct]]` vs `[[Call]]` — two internal methods

### The fundamental split

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

### Teaser reveal — why L7 throws

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

### The mental model

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

## The `[[Construct]]` protocol — step by step

When `new Dog("Rex")` executes, the engine runs `Dog.[[Construct]](["Rex"], Dog)`. Here's the full sequence:

### Step 1: OrdinaryCreateFromConstructor

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

### Step 2: Bind `this` = the fresh object

The fresh object becomes the `thisValue` passed to OrdinaryCallBindThis. Since `Dog` is an ordinary function (not an arrow), OrdinaryCallBindThis writes it into the ER's `[[ThisValue]]` slot.

Now `this` inside the constructor body refers to the fresh object.

### Step 3: Set `new.target`

The second argument to `[[Construct]]` is `newTarget` — the constructor that `new` was applied to. For a direct `new Dog()`, `newTarget = Dog`. (It differs from the running function only with `super()` calls in derived classes — covered in the class chunk.)

`new.target` is stored in the EC and accessible inside the function body.

### Step 4: Run the body

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

### Step 5: Return-value override rule

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

## Return-value override — the full picture

### Traced through the teaser

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

### Why `d1 instanceof Dog` is `false`

`instanceof` checks whether `Dog.prototype` exists in `d1`'s prototype chain. But `d1` is `{ breed: "unknown" }` — a plain object literal whose `[[Prototype]]` is `Object.prototype`. The fresh object that *had* `Dog.prototype` in its chain was discarded by the override rule.

### The common case: no explicit return

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

### Primitive returns are ignored

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

### The type gate — what counts as "Object"

The override checks `Type(result)` — the spec's type classification:

| Value | `Type(...)` | Override fires? |
|---|---|---|
| `{}`, `[]`, `function(){}`, `new Map()` | Object | Yes — replaces fresh object |
| `null` | Null | No — fresh object used |
| `undefined` (no return) | Undefined | No — fresh object used |
| `42`, `"str"`, `true`, `Symbol()`, `0n` | primitive types | No — fresh object used |

`null` is the surprising one — despite `typeof null === "object"` (the famous bug), the spec's `Type(null)` is Null, not Object. So `return null` from a constructor is ignored, and the fresh object is used.

