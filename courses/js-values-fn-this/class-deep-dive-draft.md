# 1. Class & `this` — Draft

## 1.1. Plan (teaching order)

- [x] Sub-part 1: Method shorthand — non-constructable, `this` follows the same Reference-base rule
- [x] Sub-part 2: Method extraction and `this`-loss — the class-specific shape of the problem
- [x] Sub-part 3: Field initializers — when they run, what `this` they see, arrow-field pattern
- [x] Sub-part 4: `super()` as `this`-provider in derived constructors — `this`-TDZ, the uninitialized state
- [x] Sub-part 5: Synthesis — worked example tying the pieces together

---

## 1.2. Teaser

```js
"use strict";                                         // L1
class Timer {                                         // L2
  count = 0;                                          // L3
  start() {                                           // L4
    setInterval(function () {                         // L5
      this.count++;                                   // L6
      console.log(this.count);                        // L7
    }, 1000);                                         // L8
  }                                                   // L9
  startArrow() {                                      // L10
    setInterval(() => {                               // L11
      this.count++;                                   // L12
      console.log(this.count);                        // L13
    }, 1000);                                         // L14
  }                                                   // L15
}                                                     // L16

const t = new Timer();                                // L17
t.start();                                            // L18
t.startArrow();                                       // L19
```

- **L18 (`start`):** `setInterval` invokes the callback as a plain call → Reference base = `undefined` (strict) → `this.count++` throws **TypeError**.
- **L19 (`startArrow`):** Arrow has no `this` machinery → `[[OuterEnv]]` chain walk → `startArrow`'s ER → `[[ThisValue]] = t` → `this.count++` mutates `t.count` → 1, 2, 3, …

---

## 1.3. Sub-part 1: Method shorthand — non-constructable, same `this` rule

### 1.3.1. Methods defined via shorthand have two structural differences

A method defined via **shorthand syntax** (inside a `class` body or object literal) is an ordinary function object — same `[[Call]]`, same `this`-determination pipeline. But it differs from a function declaration/expression in two ways:

> **Aside —** "Class method" in JS ≠ Python's `@classmethod`. A JS class method is an *instance* method (Python's `def bark(self):`). JS `static` is closer to Python's `@classmethod` — called on the class, `this` = the constructor. The key difference: Python's `@classmethod` descriptor *guarantees* `cls` regardless of call form; JS `static` still follows Reference-base and can lose `this` on extraction.

```js
"use strict";

// ─── Class method (method shorthand inside class body) ───
class Dog {
  bark() { return "woof"; }           // [[IsConstructor]]=false, has [[HomeObject]]
}

// ─── Function declaration ───
function bark() { return "woof"; }    // [[IsConstructor]]=true, no [[HomeObject]]

// ─── Function expression assigned to property ───
const dog = {
  bark: function() { return "woof"; } // [[IsConstructor]]=true, no [[HomeObject]]
};

// ─── Method shorthand in object literal ───
const dog2 = {
  bark() { return "woof"; }           // [[IsConstructor]]=false, has [[HomeObject]]
};
// dog2.bark and Dog.prototype.bark have identical structural properties —
// it's the shorthand syntax doing the work, not the class keyword.
```

| Property | Function declaration/expression | Class method (shorthand) |
|---|---|---|
| `[[IsConstructor]]` | `true` | `false` |
| `[[HomeObject]]` | not set | set to the class's `.prototype` object |

**`[[IsConstructor]] = false`** — you cannot `new` a method. This is the same gate arrows hit, but for a different reason: arrows lack `this` machinery entirely; methods *have* `this` machinery but are semantically "not standalone constructors." The engine enforces this at creation time based on syntactic form.

**`[[HomeObject]]`** — enables `super.method()` calls. It's a static link to the prototype object the method was defined on. Not relevant to `this` determination, but it's the other structural difference worth noting (it feeds into js-inheritance).

### 1.3.2. `this` follows the exact same Reference-base rule

```js
"use strict";                                         // L1
class Dog {                                           // L2
  name;                                               // L3
  constructor(n) { this.name = n; }                   // L4
  bark() { return this.name + " barks"; }             // L5
}                                                     // L6

const d = new Dog("Rex");                             // L7
d.bark();                                             // L8 — Reference: base=d, name="bark" → this=d → "Rex barks"

const fn = d.bark;                                    // L9 — GetValue extracts the function, Reference discarded
fn();                                                 // L10 — Reference: base=ER → this=undefined → TypeError
```

No class-specific `this` rule. L8 works because `d.bark()` is a member-expression call → Reference base = `d`. L10 fails because `fn()` is a plain identifier call → Reference base = ER → `undefined` in strict.

> **Aside —** Classes are always strict. The class body is implicitly strict regardless of surrounding code. So the sloppy-mode `globalThis` fallback never applies inside a class method — `this` is `undefined` on plain calls, never the global object.

### 1.3.3. Why non-constructable matters

```js
new d.bark();                                         // TypeError: d.bark is not a constructor
```

The engine checks `[[IsConstructor]]` before running `[[Construct]]`. Method shorthand → `false` → immediate TypeError. This is a **design-time guarantee**: methods can't accidentally be used as constructors. No `.prototype` property is created for them either (saves memory, signals intent).

---

## 1.4. Sub-part 2: Method extraction and `this`-loss

### 1.4.1. The mechanism (traced through the teaser)

Method extraction = storing a method in a variable, parameter, array slot, or any non-property-access position. The moment the next call uses that stored value, the call expression is an identifier (not a member expression on the original object) — so the Reference base is an ER, not the object.

Traced through the teaser:

```js
"use strict";                                         // L1
class EventBus {                                      // L2
  handlers = [];                                      // L3
  register(fn) { this.handlers.push(fn); }            // L4
  emit() { this.handlers.forEach(fn => fn()); }       // L5
}                                                     // L6

class Logger {                                        // L7
  prefix = "[LOG]";                                   // L8
  log() { return `${this.prefix} event fired`; }      // L9
}                                                     // L10

const bus = new EventBus();                           // L11
const logger = new Logger();                          // L12
bus.register(logger.log);                             // L13
bus.emit();                                           // L14
```

Three moments matter:

1. **L13 — extraction.** `logger.log` evaluates to a Reference (base=`logger`, name=`"log"`). But `register(fn)` is a function call — the argument position requires GetValue, which extracts the function object and **discards the Reference**. Inside `register`, `fn` is a bare function object in the parameter slot. No memory of `logger`.

2. **L4 — storage.** `this.handlers.push(fn)` stores that bare function object into the array. Still just a value — no Reference wrapper, no base.

3. **L5 — the call that loses `this`.** `.forEach(fn => fn())` iterates the array. `fn` is a parameter in the arrow's ER. The call expression `fn()` resolves `fn` via `ResolveBinding` → Reference with base = the arrow's Function ER (an Environment Record, not an object). Call operator: `IsPropertyReference`? No (base is ER) → `thisValue = undefined`.

The extraction happened at L13. The `this`-loss manifests at L5. They can be far apart — that's what makes this bug hard to spot.

**The two-path rule:**

| Call expression shape | Reference base | `thisValue` |
|---|---|---|
| `logger.log()` | `logger` (Object) | `logger` |
| `fn()` (after extraction) | Environment Record | `undefined` (strict) |

The function object is identical — same `.name`, same body, same `[[HomeObject]]`. The difference is purely in what the **call-site expression** looks like. GetValue extracts the function from the Reference at the point of assignment/passing — after that, the Reference (and its base) is gone.

### 1.4.2. Common extraction sites

Extraction isn't always as visible as `const fn = obj.method`. Every pattern below triggers GetValue on a member expression, discarding the Reference:

- **Variable assignment:** `const fn = logger.log;` → `fn()` loses `this`
- **Callback argument:** `setTimeout(logger.log, 100)` — argument passing calls GetValue
- **Destructuring:** `const { log } = logger;` — desugars to GetValue on each property
- **Higher-order functions:** `[1,2,3].forEach(logger.log)` — same as callback argument
- **Array/object storage:** `handlers.push(logger.log)` — stored as bare value; later call site determines `this`

All are the same mechanism — GetValue at the boundary, identifier-based call afterward.

> **Aside —** What about `handlers[0]()`? That *is* a property access — `IsPropertyReference` = true, base = the `handlers` array. So `this` inside `log` would be the array itself (not `undefined`). Still not `logger`, still a TypeError on `this.prefix` — but the mechanism is "wrong object as base," not "ER base → undefined." Subtly different failure path, same practical outcome.

### 1.4.3. Why classes make this worse

In pre-class code, methods on shared prototypes had the same extraction problem. Two factors make it more common and more painful with classes:

1. **Strict mode is mandatory.** Sloppy-mode constructor functions got `globalThis` as the fallback — an extracted method might silently "work" (reading `window.prefix` → `undefined`, no crash). Classes are always strict → `this = undefined` → immediate TypeError on any property access. The bug is louder, which is good — but you *must* handle it.

2. **Class methods are the primary API surface.** Classes encourage handing out instances where consumers call methods. The moment a consumer does `btn.addEventListener("click", repo.save)` or `const { save } = repo` — extraction. This is the dominant pattern in frameworks (React class components, event systems, DI containers).

### 1.4.4. The fundamental tension

Classes encourage an OOP style where behavior lives on instances (`this.method()`). But JavaScript's `this` is **per-call, not per-object** — determined by the call expression, not by where the function "belongs." Any API boundary that accepts a function (callbacks, event listeners, higher-order functions) strips the object context.

This tension is the motivation for the solutions in sub-part 3 (arrow fields) and chunk 8 (patterns & pitfalls). The mechanism is already fully explained — what remains is the toolbox for fixing it.

### 1.4.5. The flip side: capabilities per-call `this` enables

The same mechanic that breaks extraction is what enables JS to do things Java can't and Python only partially supports. The flexibility is real — but in modern application code, almost every capability has a clearer non-`this` replacement, so the actual *use* is narrow.

| Capability | JS | Python | Java | Status / when to use |
|---|---|---|---|---|
| **Method borrowing** — `fn.call(otherObj)` | Yes | Partial — unbound call works, but methods are class-bound at definition | No — `invokevirtual` baked into bytecode | **Smell in app code.** Use `Array.from(x)` / `[...x]` / rest params instead. **OK in:** library internals on array-likes, ES5 maintenance, your own stdlib-style utilities |
| **Prototype-mutation mixins** — `Object.assign(Cls.prototype, mixin)` | Yes | No — uses MRO + multiple inheritance | No — interfaces + default methods only | **Smell in most code.** Hidden deps, silent name collisions, broken `super`. **OK in:** documented framework internals, polyfills, test scaffolding. **Prefer:** composition, standalone functions, class factory mixins |
| **Generic `this`-shape algorithms** — `function f() { return this.length }` | Yes | Limited — duck typing exists but methods are class-bound | No | **Smell in app code.** Implicit-first-argument via `this` is cute but unclear. **OK in:** protocol utilities meant to be installed on prototypes, library code targeting any array-like / iterable. **Prefer:** explicit first argument (`f(x)`) like `Array.from`, `Object.keys` |
| **Free method extraction** — `const fn = obj.method` (bare function passed around) | Yes — automatic | Yes — but Python's bound-method wrapper makes it less common | No — method refs are wrappers, not bare functions | **Mixed.** Cause of `this`-loss bug, but enables functional pipelines. **OK when:** function is pure / arrow-bound / explicitly `.bind`-ed. **Smell when:** extracting a `this`-using method without binding |
| **Run-time prototype reassignment** — `Object.setPrototypeOf(obj, NewProto)` | Yes | Limited (`__class__` reassignment, rare) | No | **Strong smell.** Wrecks engine optimizations, hard to reason about. **OK in:** reflection-heavy library code, certain proxy/wrapper patterns, type-mutating test tools. **Almost never** in app code |

**The pattern.** Per-call `this` is a sharp generic-dispatch tool. Its real practical use today is narrow — it lives in **library/framework internals**, **polyfills/shims**, and **reading/maintaining old code** that predates modern alternatives. For application code, modern JS has converged on:

- Explicit first arguments over `this`-dispatch (`Array.from(x)`, `Object.keys(o)`)
- Composition over mixins
- Spread / `Array.from` / rest params over `.call`-borrowing
- Arrow fields or explicit `.bind` over hoping `this` survives extraction

The capabilities exist to enable libraries — not to be reached for in everyday code.

---

## 1.5. Sub-part 3: Field initializers — when they run, what `this` they see

### 1.5.1. What a field is, mechanically

An instance field declares a property that the engine assigns **per instance** during construction. (Static fields are per-class, run at class evaluation time — not relevant to `this`-binding here.) The syntax:

```js
class Foo {                                           // L1
  count = 0;                                          // L2 — public instance field
  #secret = "hidden";                                 // L3 — private instance field
  static name = "Foo";                                // L4 — static field (set on class, not instances)
  constructor() {                                     // L5
    console.log(this.count);                          // L6 — 0 (field already assigned)
    this.count = 99;                                  // L7 — constructor body overwrites
  }                                                   // L8
}                                                     // L9
```

is sugar for: "during `[[Construct]]`, after `this` exists, run `this.count = 0` (L2) and `this.#secret = "hidden"` (L3), then run the constructor body (L5–L8)." The static field (L4) runs once when the class is evaluated, not per instance. Instance fields are not stored on the prototype — each instance gets its own copy. This is the structural opposite of methods (which live on the prototype, shared).

### 1.5.2. When initializers run

Each instance field has its own **initializer** — the right-hand side expression (`0` in `count = 0`, `"hidden"` in `#secret = "hidden"`). Internally, the engine wraps each initializer in a separate function that it calls with `this` = the fresh instance. Fields without an initializer (`name;`) default to `undefined`.

In a **base class** (no `extends`), these initializers run at the start of the constructor, immediately after `this` is bound to the fresh object:

```
[[Construct]] for base class Foo:
  1. OrdinaryCreateFromConstructor → fresh object, [[Prototype]] = Foo.prototype
  2. Bind this = fresh object
  3. Run instance field initializers in source order  ← L2, L3 run here
  4. Run constructor body                             ← L5–L8 run here
  5. Return-value override rule

(L4 — static field — runs once at class evaluation time, not during [[Construct]])
```

L6 sees `0` because the field initializer (L2, step 3) ran before the constructor body (step 4). The constructor can read and overwrite fields that already exist on `this`.

The initializer expression is evaluated **for each new instance** — the right-hand side is not shared. `count = []` gives every instance its own array.

In a **derived class** (`class Bar extends Foo`), fields run **after `super()` returns**. This shifts everything; we'll cover it in sub-part 4.

### 1.5.3. What `this` is inside an initializer

Field initializers run as part of the constructor's execution context. So `this` inside an initializer is the same `this` the constructor sees — the fresh instance.

```js
"use strict";                                         // L1
class Counter {                                       // L2
  start = 0;                                          // L3
  current = this.start;                               // L4 — this = fresh instance, this.start = 0
  label = `counter-${++Counter.id}`;                  // L5 — runs per instance, increments static id
  static id = 0;                                      // L6
}                                                     // L7

const a = new Counter();                              // L8 → { start: 0, current: 0, label: "counter-1" }
const b = new Counter();                              // L9 → { start: 0, current: 0, label: "counter-2" }
```

L4 demonstrates that earlier fields are visible to later ones — they run in source order, and each assigns to `this` before the next runs. L5 runs per instance, so `Counter.id` increments each time.

> **Aside —** Spec mechanics: each field is internally a function (the "initializer function") whose `[[HomeObject]]` is the class's `.prototype` (for instance fields) or the class itself (for static fields). The constructor calls each initializer function via a special `[[Call]]` that uses the fresh instance as `thisValue`. So inside an initializer, `this` follows the same `[[ThisValue]]`-slot mechanism as a method body — just invoked by the engine, not your code.

### 1.5.4. The arrow-field pattern — solving `this`-loss structurally

Recall from sub-part 2: extracting a prototype method (passing it as a callback, storing in a variable) loses `this` because the subsequent call is an identifier call → ER base → `undefined`. The combination of two facts gives a structural fix:

1. Field initializers run with `this` = the instance.
2. Arrow functions capture `this` lexically — once captured, no invocation path can override it.

**The problem (method on prototype):**

```js
"use strict";                                         // L1
class Logger {                                        // L2
  prefix = "[LOG]";                                   // L3
  log(msg) { return `${this.prefix} ${msg}`; }        // L4 — method on Logger.prototype
}                                                     // L5

const l = new Logger();                               // L6
const fn = l.log;                                     // L7 — extraction: GetValue discards Reference
fn("hi");                                             // L8 → TypeError: Cannot read properties of undefined (reading 'prefix')
```

L8: `fn` is an identifier → ER base → `this = undefined` → `undefined.prefix` throws.

**The fix (arrow field):**

```js
"use strict";                                         // L1
class Logger {                                        // L2
  prefix = "[LOG]";                                   // L3
  log = (msg) => `${this.prefix} ${msg}`;             // L4 — arrow captures this = instance
}                                                     // L5

const l = new Logger();                               // L6
const fn = l.log;                                     // L7 — extraction
fn("hi");                                             // L8 → "[LOG] hi" — works!
setTimeout(l.log, 100);                               // L9 → still works as callback
```

L4 trace: when `new Logger()` runs, the `log` initializer evaluates `(msg) => ...`. The arrow's `[[OuterEnv]]` is captured at that moment, pointing at the constructor's ER, whose `[[ThisValue]]` is the fresh instance. From then on, `this` inside `log` always resolves to that instance via the chain walk — regardless of how `log` is called.

### 1.5.5. Tradeoff: per-instance copies vs prototype sharing

Arrow fields aren't free — they trade memory for safety:

| Aspect | Method (prototype) | Arrow field (per-instance) |
|---|---|---|
| Storage | One function on `Foo.prototype`, shared by all instances | One function object per instance |
| `this`-binding | Per-call (via Reference base) | Per-instance (lexically captured) |
| Survives extraction? | No — extraction loses `this` | Yes — `this` is locked |
| Memory | O(1) per class | O(n) per n instances |
| Available on prototype? | Yes — `Foo.prototype.method` | No — it's an instance property |
| `super` works? | Yes (has `[[HomeObject]]`) | No (arrows lack `[[HomeObject]]`) |
| Overridable in subclass? | Yes — subclass method shadows prototype's | Awkward — must override as field too |

**When to reach for which:**

- **Method (default).** Most class members. Sharing on the prototype is the right default — it's why prototypes exist. Use when callers will invoke as `obj.method()` (the typical OOP pattern).
- **Arrow field.** When the method *will* be extracted — passed as a callback, attached as an event handler, stored in a data structure. Common in React class components (`handleClick = () => {...}`) and any code that hands methods to external systems. Pay the per-instance cost to get structural `this`-locking.
- **`bind` in constructor.** Older alternative — `this.method = this.method.bind(this);` in the constructor body. Same per-instance cost, more boilerplate, no structural advantage over arrow fields. Predates class fields. Use only if targeting environments without field syntax.

> **Aside —** "Class fields" was a long-running proposal (TC39 stage 3 for years before reaching stage 4 in ES2022). Earlier code uses constructor-body assignment (`this.count = 0;` in `constructor()`) or Babel/TypeScript with the proposal flag. Modern engines (Node 12+, all current browsers) support it natively.

### 1.5.6. What goes wrong if you reach for arrow fields by default

Two failure modes worth flagging:

```js
class Animal {                                        // L1
  speak = () => "generic sound";                      // L2 — arrow field
}                                                     // L3
class Dog extends Animal {                            // L4
  speak() { return "woof"; }                          // L5 — method on prototype
}                                                     // L6

const d = new Dog();                                  // L7
d.speak();                                            // L8 → "generic sound"  ← surprising!
```

L8 trace: `Dog` inherits from `Animal`. After `super()`, `Animal`'s field initializer runs and assigns the arrow to `this.speak` — directly on the instance. `Dog.prototype.speak` exists too, but property lookup finds the instance property first. The subclass method is **shadowed** by the parent's instance field.

Second failure mode: arrow fields can't call `super`. They lack `[[HomeObject]]`, so `super.method()` references nothing.

```js
class Child extends Parent {                          // L1
  greet = () => super.greet();                        // L2 — SyntaxError
}                                                     // L3
```

The parser actually rejects this as a syntax error in some forms; in others it throws at runtime. Either way: if you need `super`, use a method, not an arrow field.

---

## 1.6. Sub-part 4: `super()` as `this`-provider in derived constructors

### 1.6.1. The problem: where does `this` come from in a derived class?

In a base class, `[[Construct]]` creates the fresh object *before* the constructor body runs (step 1 in the sequence from sub-part 3). The constructor's `[[ThisValue]]` is set immediately — fields and the body both see `this` from the start.

A derived class **cannot do this**. The fresh object must have the correct `[[Prototype]]` — which depends on the *most-derived* constructor's `.prototype`. But the engine enters the *base* constructor first (via `super()` calls up the chain). So the object can't be created until the base constructor runs.

This creates a structural asymmetry:

| | Base class | Derived class |
|---|---|---|
| Who creates the object? | `[[Construct]]` (before constructor body) | The **base** constructor (via `super()` chain) |
| When is `this` available? | Immediately | Only after `super()` returns |
| What happens before `this` exists? | N/A — it always exists | `this` is in an **uninitialized** state |

### 1.6.2. The `this`-TDZ: uninitialized `[[ThisValue]]`

"TDZ" (Temporal Dead Zone) is borrowed from `let`/`const` — the slot exists but accessing it before initialization throws. The same concept applies to `this` in a derived constructor:

```js
"use strict";                                         // L1
class Base {                                          // L2
  constructor() {                                     // L3
    console.log("Base: this exists", this);           // L4
  }                                                   // L5
}                                                     // L6

class Derived extends Base {                          // L7
  constructor() {                                     // L8
    console.log(this);                                // L9 — ReferenceError!
    super();                                          // L10
    console.log(this);                                // L11 — works: this = fresh object
  }                                                   // L12
}                                                     // L13

new Derived();                                        // L14
```

L9 throws `ReferenceError: Must call super constructor in derived class before accessing 'this' or returning from derived constructor`. The `[[ThisValue]]` slot in the derived constructor's Function ER is **uninitialized** — the engine checks this on every `this` reference and throws if it hasn't been set yet.

L10 (`super()`) is what initializes it. After `super()` returns, `this` is the fresh object that the base constructor created.

### 1.6.3. What `super()` actually does — the full sequence

`super()` in a derived constructor is not just "call the parent constructor." It's the mechanism that **provides `this`** to the derived constructor. The sequence:

```
Derived constructor's [[Construct]] (called via `new Derived()`):
  1. DO NOT create an object (unlike base class)
  2. Set [[ThisValue]] = UNINITIALIZED in the constructor's Function ER
  3. Enter constructor body

  ... code before super() runs with this = UNINITIALIZED ...

  4. super() encountered:
     a. Resolve [[ConstructorToCall]] = Base (via [[HomeObject]].[[Prototype]])
     b. Call Base.[[Construct]](args, newTarget=Derived)
        i.   OrdinaryCreateFromConstructor(Derived) → fresh object
             [[Prototype]] = Derived.prototype (uses newTarget, not Base!)
        ii.  Bind this = fresh object in Base's ER
        iii. Run Base's field initializers
        iv.  Run Base's constructor body
        v.   Return fresh object (unless explicit return override)
     c. Take the returned object
     d. BindThisValue(derivedConstructor's ER, returnedObject)
        → [[ThisValue]] slot goes from UNINITIALIZED → the object

  5. this is now live — Derived's field initializers run here
  6. Rest of Derived's constructor body runs
  7. Implicit return this (or explicit return override)
```

Key insight: `newTarget` in step 4.b.i is `Derived`, not `Base`. That's why the fresh object gets `[[Prototype]] = Derived.prototype` even though `Base`'s `[[Construct]]` creates it. The `new.target` value propagates down the `super()` chain.

### 1.6.4. Field initializer ordering in derived classes

This is where sub-part 3's "fields run at the start of the constructor" gets refined. The full picture:

```js
"use strict";                                         // L1
class Base {                                          // L2
  baseField = console.log("Base field");              // L3
  constructor() {                                     // L4
    console.log("Base constructor body");             // L5
  }                                                   // L6
}                                                     // L7

class Derived extends Base {                          // L8
  derivedField = console.log("Derived field");        // L9
  constructor() {                                     // L10
    console.log("Before super");                      // L11
    super();                                          // L12
    console.log("After super");                       // L13
  }                                                   // L14
}                                                     // L15

new Derived();                                        // L16
```

Output:
```
Before super
Base field
Base constructor body
Derived field
After super
```

Trace:
1. `new Derived()` → enter Derived constructor, `this` = UNINITIALIZED
2. L11 logs (no `this` access, so no error)
3. L12 `super()` → enters Base's `[[Construct]]`:
   - Creates fresh object (with `[[Prototype]] = Derived.prototype`)
   - Runs Base's field initializers → L3 logs "Base field"
   - Runs Base's constructor body → L5 logs "Base constructor body"
   - Returns the object
4. Back in Derived: `this` is now initialized to the returned object
5. Derived's field initializers run → L9 logs "Derived field"
6. L13 logs "After super"

**The rule:** Base fields → Base body → (super returns) → Derived fields → Derived body (rest). Each class's fields run before its own constructor body, but the entire base construction completes before derived fields start.

### 1.6.5. What you can and cannot do before `super()`

The `this`-TDZ is enforced on any `this` reference — but not all code requires `this`:

```js
class Derived extends Base {                          // L1
  constructor(config) {                               // L2
    // ✅ OK — no this access:
    const validated = validate(config);               // L3
    console.log("setting up");                        // L4
    if (!validated) throw new Error("bad config");    // L5

    super(validated);                                 // L6

    // ✅ OK — this is live:
    this.config = validated;                          // L7
  }                                                   // L8
}                                                     // L9
```

You can run arbitrary code before `super()` — compute arguments, validate, throw early — as long as you don't touch `this` or `super.property`. This is intentional: it lets derived constructors prepare arguments for the base constructor.

**What throws before `super()`:**
- `this.anything` — ReferenceError
- `super.method()` — ReferenceError (needs `this` to dispatch on)
- Implicit `return;` without calling `super()` — ReferenceError (the constructor must provide `this`)

**What's fine before `super()`:**
- Local variables, function calls, conditionals, `throw`
- `arguments`, closures, anything that doesn't reference `this`

### 1.6.6. The "must call super" rule

A derived constructor **must** call `super()` before it returns (unless it explicitly returns an object — the return-value override from chunk 6). If you omit `super()` entirely:

```js
class Broken extends Base {                           // L1
  constructor() {                                     // L2
    // no super() call                                // L3
  }                                                   // L4 — implicit return this → ReferenceError!
}                                                     // L5

new Broken();                                         // L6
```

The implicit `return this` at L4 tries to read `[[ThisValue]]` — still UNINITIALIZED → ReferenceError. The only escape hatch: `return someObject;` (explicit non-`this` return), which bypasses the `this` slot entirely. This is the same return-value override rule from chunk 6, and it's the only way a derived constructor can avoid calling `super()`.

### 1.6.7. Default constructors — when you omit `constructor`

If you don't write a constructor, the engine synthesizes one. The synthesized body depends on whether the class extends:

```js
// Base class (no extends) — synthesized:
constructor() {}

// Derived class (extends X) — synthesized:
constructor(...args) { super(...args); }
```

The derived default forwards every argument to the parent — that's why omitting the constructor on a subclass still works when the parent expects arguments:

```js
class Animal {                                        // L1
  constructor(name) { this.name = name; }             // L2
}                                                     // L3
class Dog extends Animal {}                           // L4 — synthesized: constructor(...args) { super(...args); }
const d = new Dog("Rex");                             // L5 → d.name = "Rex"
```

The `...args` spread preserves arity. If the synthesized form were `constructor() { super(); }`, arguments would silently drop (`d.name` would be `undefined`). Argument forwarding is transparent by default.

The synthesis is purely a parser-level convenience. By the time `[[Construct]]` runs, there's a real function in place — **same `[[ThisValue]] = UNINITIALIZED` start, same `super()` providing `this`, same field-initializer ordering.** Field initializers still run as usual; the synthesized constructor just has nothing else in its body.

This is also why the `this`-TDZ surprises people: most simple subclasses never trip it, because the synthesized `super(...args)` satisfies the rule before any user code runs. The TDZ only bites when you write your own derived constructor and access `this` before the `super()` call.

**Write an explicit derived constructor only when you need to:**
- Do something *before* `super()` (validate, transform, throw early)
- Do something *after* `super()` (set extra fields, register the instance)

If neither applies, omit it. The synthesized default is identical to what you'd write, and the absence makes intent clearer.

### 1.6.8. Why this design? (motivation)

The `this`-TDZ exists because of a real constraint: the engine can't create the instance until it knows the full prototype chain setup. In single inheritance this seems over-engineered — but consider:

1. **The base constructor might have side effects** that depend on the object existing (assigning fields, registering in a global map). Those must happen on the *final* object, not a temporary.
2. **`new.target` propagation** ensures the object gets the right `[[Prototype]]` even though the base creates it. Without this, `instanceof` would break.
3. **The TDZ catches real bugs** — accessing `this` before the object is fully set up (base fields assigned, base constructor logic run) would read uninitialized properties. The ReferenceError is better than silent `undefined`.

The Python comparison: Python's `__init__` receives `self` already created (by `__new__`). There's no TDZ — `self` is always available. But Python also can't enforce "base init ran first" — you can forget `super().__init__()` and get a half-initialized object silently. JS chose strictness over convenience here.


---

## 1.7. Sub-part 5: Worked synthesis

A single annotated example tracing every mechanism from sub-parts 1–4 through one construction. Each commented line marks which mechanism is firing.

```js
"use strict";                                                          // L1
class Component {                                                      // L2
  // ─── Instance field — runs in source order during [[Construct]] ─── // L3
  state = { count: 0 };                                                // L4 — base field initializer

  // ─── Arrow field — fresh function PER INSTANCE; this captured lexically ─── // L5
  log = () => `[${this.constructor.name}] count=${this.state.count}`;  // L6 — arrow field (n instances → n function objects)

  // ─── Method shorthand — ONE function on prototype, shared by all instances ─── // L7
  tick() {                                                             // L8 — non-constructable; lives on Component.prototype
    this.state.count++;                                                // L9 — Reference base = this
    return this;                                                       // L10
  }                                                                    // L11

  constructor(label) {                                                 // L12
    this.label = label;                                                // L13 — body runs after fields
  }                                                                    // L14
}                                                                      // L15

class Counter extends Component {                                      // L16
  // ─── Derived field — runs AFTER super() returns ─── // L17
  step = 1;                                                            // L18 — derived field initializer

  // No constructor written → synthesized: constructor(...args) { super(...args); }
  // Argument forwarding handles the "label" parameter automatically.
}                                                                      // L21

class FastCounter extends Counter {                                    // L22
  step = 10;                                                           // L23 — overrides Counter's step

  constructor(label) {                                                 // L24
    // ─── Pre-super code: this is UNINITIALIZED here ─── // L25
    if (!label) throw new Error("label required");                     // L26 — OK, no this access

    super(label);                                                      // L27 — provides this; runs full Counter→Component chain

    // ─── Post-super: this is live ─── // L28
    this.tick();                                                       // L29 — Reference base = this → tick runs with this=instance
  }                                                                    // L30
}                                                                      // L31

const fc = new FastCounter("fast");                                    // L32

// ─── Method extraction → this-loss ─── // L33
const safe = fc.log;                                                   // L34 — log is arrow field, this is locked
console.log(safe());                                                   // L35 — works! "[FastCounter] count=1"

const lost = fc.tick;                                                  // L36 — tick is a method, not arrow field
lost();                                                                // L37 — TypeError: Cannot read properties of undefined (reading 'state')
```

### 1.7.1. Trace through L32 — the construction sequence

`new FastCounter("fast")` triggers a chain of `[[Construct]]` calls. Following the mechanisms:

```
1. FastCounter.[[Construct]]("fast", newTarget=FastCounter):
   - [[ThisValue]] = UNINITIALIZED (sub-part 4: this-TDZ)
   - Enter L24 constructor body
   - L26: validate label — OK, no this access
   - L27 super("fast") → Counter.[[Construct]]("fast", newTarget=FastCounter):

2. Counter.[[Construct]]("fast", newTarget=FastCounter):
   - Synthesized constructor: (...args) { super(...args); }
   - [[ThisValue]] = UNINITIALIZED
   - super("fast") → Component.[[Construct]]("fast", newTarget=FastCounter):

3. Component.[[Construct]]("fast", newTarget=FastCounter):
   - OrdinaryCreateFromConstructor(newTarget=FastCounter)
     → fresh object with [[Prototype]] = FastCounter.prototype  ← key: newTarget, not Component
   - BindThisValue: Component's ER [[ThisValue]] = fresh object
   - Run Component's field initializers in source order:
     • L4: this.state = { count: 0 }      ← own property on the fresh object
     • L6: this.log = (msg) => ...        ← arrow created NOW; [[OuterEnv]] captures
                                            Component's ER, whose [[ThisValue]] = fresh object
                                            (sub-part 3: arrow field locks this here)
   - Run Component's constructor body:
     • L13: this.label = "fast"
   - Return fresh object

4. Back in Counter (step 2):
   - BindThisValue: Counter's ER [[ThisValue]] = fresh object  ← same object
   - Run Counter's field initializers:
     • L18: this.step = 1                 ← own property assigned
   - Synthesized body has nothing else
   - Return fresh object

5. Back in FastCounter (step 1):
   - BindThisValue: FastCounter's ER [[ThisValue]] = fresh object  ← same object, now in scope
   - Run FastCounter's field initializers:
     • L23: this.step = 10                ← OVERWRITES the 1 written at L18 (sub-part 4: derived fields run after super)
   - Run FastCounter's constructor body:
     • L29: this.tick()                   ← Reference base = this (the instance) → tick runs with this=instance
                                            (sub-part 1: method shorthand follows Reference-base rule)
                                            tick body: this.state.count++ → state.count = 1
   - Return fresh object

Final fc:
  { state: { count: 1 }, log: <arrow>, label: "fast", step: 10 }
  [[Prototype]] → FastCounter.prototype → Counter.prototype → Component.prototype → Object.prototype
```

### 1.7.2. Trace through L34–L37 — extraction outcomes diverge

Two extractions, opposite outcomes — same `this`-loss mechanism, different protections:

```
L34: const safe = fc.log
  - fc.log is a Reference (base=fc, name="log")
  - GetValue extracts the arrow function, discards Reference
  - safe = the arrow function object

L35: safe()
  - safe() is identifier call → Reference base = ER → thisValue = undefined
  - But arrow ignores thisValue! [[ThisValue]] resolved via [[OuterEnv]] chain walk
  - Walks to Component's ER (captured at L6) → finds fresh object (= fc)
  - Returns "[FastCounter] count=1"
  - this.constructor.name reads FastCounter.name (the actual class via prototype walk)
  - WORKS — sub-part 3: arrow field structurally locks this

L36: const lost = fc.tick
  - fc.tick is a Reference (base=fc, name="tick")
  - GetValue extracts the function from FastCounter.prototype's chain
    (lookup walks FastCounter.prototype → Counter.prototype → Component.prototype → finds tick)
  - lost = the method's function object (no [[BoundThis]], not an arrow)

L37: lost()
  - identifier call → Reference base = ER → thisValue = undefined (strict)
  - Method's body executes with this = undefined
  - L9: this.state.count++ → undefined.state → TypeError
  - FAILS — sub-part 2: method extraction loses this
```

### 1.7.3. What the example demonstrates

Reading top-to-bottom, each annotation maps to one earlier mechanism:

| Line | Mechanism | From |
|---|---|---|
| L4 | Field initializer runs during `[[Construct]]`, source order | Sub-part 3 |
| L6 | Arrow field captures `this` at construction time | Sub-part 3 |
| L8 | Method shorthand → non-constructable, lives on prototype | Sub-part 1 |
| L9 | Method body's `this` from Reference base at call site | Sub-part 1 |
| L18 → L23 | Derived field overwrites base field on same object | Sub-part 4 |
| L26 | Pre-`super()` code is fine without `this` access | Sub-part 4 |
| L27 | `super()` provides `this` to derived constructor | Sub-part 4 |
| L29 | Method call inside constructor — Reference base = `this` | Sub-part 1 |
| L34–L35 | Arrow field survives extraction | Sub-part 3 |
| L36–L37 | Prototype method loses `this` on extraction | Sub-part 2 |

The single `[[Construct]]` chain ties all four sub-parts together: object creation flows up the chain, `this` flows back down, fields run at each level in a strict order, and the final object is what every method, arrow, and extraction interacts with afterward.
