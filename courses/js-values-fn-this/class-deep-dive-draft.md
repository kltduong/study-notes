# Class & `this` — Draft

## Plan (teaching order)

- [x] Sub-part 1: Class (shorthand) methods — non-constructable, `this` follows the same Reference-base rule
- [x] Sub-part 2: Method extraction and `this`-loss — the class-specific shape of the problem
- [x] Sub-part 3: Field initializers — when they run, what `this` they see, arrow-field pattern
- [ ] Sub-part 4: `super()` as `this`-provider in derived constructors — `this`-TDZ, the uninitialized state
- [ ] Sub-part 5: Synthesis — worked example tying the pieces together

---

## Teaser

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

## Sub-part 1: Class (shorthand) methods — non-constructable, same `this` rule

### Methods are ordinary functions with two structural differences

A class method (defined via method shorthand inside a `class` body) is an ordinary function object — same `[[Call]]`, same `this`-determination pipeline. But it differs from a function declaration/expression in two ways:

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

### `this` follows the exact same Reference-base rule

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

### Why non-constructable matters

```js
new d.bark();                                         // TypeError: d.bark is not a constructor
```

The engine checks `[[IsConstructor]]` before running `[[Construct]]`. Method shorthand → `false` → immediate TypeError. This is a **design-time guarantee**: methods can't accidentally be used as constructors. No `.prototype` property is created for them either (saves memory, signals intent).

---

## Sub-part 2: Method extraction and `this`-loss

### The mechanism (traced through the teaser)

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

### Common extraction sites

Extraction isn't always as visible as `const fn = obj.method`. Every pattern below triggers GetValue on a member expression, discarding the Reference:

- **Variable assignment:** `const fn = logger.log;` → `fn()` loses `this`
- **Callback argument:** `setTimeout(logger.log, 100)` — argument passing calls GetValue
- **Destructuring:** `const { log } = logger;` — desugars to GetValue on each property
- **Higher-order functions:** `[1,2,3].forEach(logger.log)` — same as callback argument
- **Array/object storage:** `handlers.push(logger.log)` — stored as bare value; later call site determines `this`

All are the same mechanism — GetValue at the boundary, identifier-based call afterward.

> **Aside —** What about `handlers[0]()`? That *is* a property access — `IsPropertyReference` = true, base = the `handlers` array. So `this` inside `log` would be the array itself (not `undefined`). Still not `logger`, still a TypeError on `this.prefix` — but the mechanism is "wrong object as base," not "ER base → undefined." Subtly different failure path, same practical outcome.

### Why classes make this worse

In pre-class code, methods on shared prototypes had the same extraction problem. Two factors make it more common and more painful with classes:

1. **Strict mode is mandatory.** Sloppy-mode constructor functions got `globalThis` as the fallback — an extracted method might silently "work" (reading `window.prefix` → `undefined`, no crash). Classes are always strict → `this = undefined` → immediate TypeError on any property access. The bug is louder, which is good — but you *must* handle it.

2. **Class methods are the primary API surface.** Classes encourage handing out instances where consumers call methods. The moment a consumer does `btn.addEventListener("click", repo.save)` or `const { save } = repo` — extraction. This is the dominant pattern in frameworks (React class components, event systems, DI containers).

### The fundamental tension

Classes encourage an OOP style where behavior lives on instances (`this.method()`). But JavaScript's `this` is **per-call, not per-object** — determined by the call expression, not by where the function "belongs." Any API boundary that accepts a function (callbacks, event listeners, higher-order functions) strips the object context.

This tension is the motivation for the solutions in sub-part 3 (arrow fields) and chunk 8 (patterns & pitfalls). The mechanism is already fully explained — what remains is the toolbox for fixing it.

### The flip side: capabilities per-call `this` enables

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

## Sub-part 3: Field initializers — when they run, what `this` they see

### What a field is, mechanically

A class field is a **per-instance property assignment** that the engine performs as part of construction. The syntax:

```js
class Foo {
  count = 0;            // public field with initializer
  #secret = "hidden";   // private field
  static name = "Foo";  // static field (set on the class itself, not instances)
}
```

is sugar for: "during `[[Construct]]`, after `this` exists, run `this.count = 0` and `this.#secret = "hidden"`." Fields are not stored on the prototype — each instance gets its own copy. This is the structural opposite of methods (which live on the prototype, shared).

### When initializers run

In a **base class** (no `extends`), field initializers run at the start of the constructor, immediately after `this` is bound to the fresh object:

```
[[Construct]] for base class Foo:
  1. OrdinaryCreateFromConstructor → fresh object, [[Prototype]] = Foo.prototype
  2. Bind this = fresh object
  3. Run all field initializers in source order  ← fields run here
  4. Run constructor body
  5. Return-value override rule
```

The initializer expression is evaluated **for each new instance** — the right-hand side is not shared. `count = []` gives every instance its own array.

In a **derived class** (`class Bar extends Foo`), fields run **after `super()` returns**. This shifts everything; we'll cover it in sub-part 4.

### What `this` is inside an initializer

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

### The arrow-field pattern — solving `this`-loss structurally

The combination of two facts unlocks the pattern:

1. Field initializers run with `this` = the instance.
2. Arrow functions capture `this` lexically — once captured, no invocation path can override it.

So if you assign an arrow to a field, the arrow captures the instance as its `this`, **per instance**, at construction time:

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

### Tradeoff: per-instance copies vs prototype sharing

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

### What goes wrong if you reach for arrow fields by default

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
