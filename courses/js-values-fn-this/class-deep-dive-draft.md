# Class & `this` — Draft

## Plan (teaching order)

- [x] Sub-part 1: Class methods — non-constructable, `this` follows the same Reference-base rule
- [x] Sub-part 2: Method extraction and `this`-loss — the class-specific shape of the problem
- [ ] Sub-part 3: Field initializers — when they run, what `this` they see, arrow-field pattern
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

## Sub-part 1: Class methods — non-constructable, same `this` rule

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

### Where extraction happens in class code

Extraction isn't always obvious. Every pattern below stores the method as a bare value, losing the object base:

```js
"use strict";                                         // L1
class Logger {                                        // L2
  prefix = "[LOG]";                                   // L3
  log() { return `${this.prefix} message`; }          // L4
}                                                     // L5
const logger = new Logger();                          // L6

// Pattern 1: explicit variable assignment
const fn = logger.log;                                // L7
fn();                                                 // L8 → TypeError

// Pattern 2: passed as callback argument
setTimeout(logger.log, 100);                          // L9 → TypeError inside

// Pattern 3: stored in a data structure
const handlers = [logger.log];                        // L10
handlers[0]();                                        // L11 → TypeError

// Pattern 4: destructuring
const { log } = logger;                               // L12
log();                                                // L13 → TypeError

// Pattern 5: higher-order — passed to .map/.forEach/etc.
[1, 2, 3].forEach(logger.log);                        // L14 → TypeError
```

All five are the same mechanism: `logger.log` evaluates to a Reference (base=`logger`, name=`"log"`), then GetValue extracts the function object for assignment/passing, discarding the Reference. The subsequent call has no memory of `logger`.

> **Aside —** Pattern 3 (`handlers[0]()`) is subtly different from the others syntactically — it *is* a property access (element access on the array). But the base of that Reference is the `handlers` array, not `logger`. So `this` inside `log` would be the array (in sloppy mode) or `undefined` after strict-mode coercion rules apply. Either way, not `logger`.

### Why classes make this worse than plain objects

In pre-class code, methods lived on shared prototypes too — but two factors make extraction more common and more painful with classes:

1. **Strict mode is mandatory.** Old-style constructor functions in sloppy mode got `globalThis` as the fallback — the method might accidentally work (reading `window.prefix` returns `undefined`, not a TypeError). Classes are always strict → immediate TypeError on `this.anything`. The bug is louder, which is good — but it means you *must* handle it.

2. **Class methods are the primary API surface.** With classes, you hand out instances and consumers call methods. The moment a consumer does `const { save } = repo;` or `btn.addEventListener("click", repo.save)` — extraction. This is the dominant pattern in frameworks (React class components, event systems, DI containers).

### The fundamental tension

Classes encourage an OOP style where behavior lives on instances (`this.method()`). But JavaScript's `this` is **per-call, not per-object** — it's determined by the call expression, not by where the function "belongs." Any API boundary that accepts a function (callbacks, event listeners, higher-order functions) strips the object context.

This tension is the motivation for the solutions in sub-part 3 (arrow fields) and chunk 8 (patterns & pitfalls). The mechanism is already fully explained — what remains is the toolbox for fixing it.
