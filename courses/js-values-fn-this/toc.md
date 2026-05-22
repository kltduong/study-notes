# JS Values, Functions & `this` — Course TOC

Progress tracker for the course.

## Calibration

**Starting point:** Knows EC/ER internals from js-vars-scope (Function ER, `[[ThisValue]]` slot, `[[OuterEnv]]` chain, creation/execution phases). Comfortable with function declarations, arrow functions, and `call`/`apply`/`bind` in daily use. Fuzzy on the spec-level mechanism — the Reference type, how the engine extracts `this` from a call expression, why certain patterns lose `this`.

**Goals:**

1. One-rule mental model for `this` determination (Reference base) + know exactly which mechanisms override it.
2. Understand functions as a call protocol — not just "run the body" but the full sequence from expression evaluation to EC creation.
3. Confidently predict `this` in any code without memorizing case lists.

**Planned arc:**

1. **Values & memory model** — primitive vs object, slot holds value directly (primitives) vs holds a reference (objects), copy semantics on assignment/argument passing, identity (`===` compares slot content). Foundation for understanding what `this` *is* (a reference in a slot).
2. **The Reference type** — what the engine produces when evaluating member expressions; base + name as a structured intermediate value. The foundation everything else derives from.
3. **`this` determination** — the one rule (Reference base → `[[ThisValue]]`) and why plain calls yield `undefined`. Strict vs sloppy coercion.
4. **Explicit overrides: `call`, `apply`, `bind`** — mechanisms that bypass the Reference-base default. Partial application, the bound-function wrapper.
5. **Arrow functions** — structurally different (no `[[ThisValue]]`, no `arguments`, no `new.target`). Lexical `this` via `[[OuterEnv]]` chain walk. Why they exist (the `self = this` problem).
6. **Constructor calls (`new`)** — the object-creation protocol, `new.target`, return-value override rule, why `class` constructors are `new`-only.
7. **Class & `this`** — field initializers, method `this`-loss, `super()` and `this`-TDZ in derived constructors. Call-protocol aspects only (inheritance mechanics stay in js-inheritance).
8. **Patterns & pitfalls** — callbacks losing `this`, event handlers, method extraction, historical patterns (`self`/`that`), modern solutions (arrow fields, `bind` in constructor).

**Pacing notes:** Chunk 1 is the new foundation — value semantics and memory layout. Chunks 2–3 are the core `this` machinery — the Reference type is the new spec-level concept; spend time there. Chunk 4 is moderate. Chunks 5–6 build on the foundation with clear derivations. Chunks 7–8 are application/synthesis. Probe "right conclusion, wrong mechanism" weakness — `this` is a prime area for coincidence-based reasoning.

**Dependencies:** js-vars-scope (EC, Function ER, `[[ThisValue]]` slot, `[[OuterEnv]]` chain). Feeds into js-inheritance (method dispatch, constructor `this`).

## Progress

- [x] **Values & memory model** — primitive vs object (two categories), slot holds value directly vs holds a heap reference, copy semantics (assignment/passing copy the slot), identity (`===` compares slot content), implications for mutation and aliasing
      📊 solid
- [x] **The Reference type** — spec-level Reference Record, base/name/strict fields, how member expressions produce References, how plain identifiers produce References with ER base
      📊 solid
      🔧 Refinements: — Said "ReferenceError" twice when the failure is property access on `undefined` (TypeError). ReferenceError is for unresolvable identifiers only.
- [x] **`this` determination** — the single rule: extract base from Reference at call site → `[[ThisValue]]`; ER base → `undefined`; strict vs sloppy coercion to `globalThis`
      📊 solid
      🔧 Refinements: — Assumed optional chaining `?.()` consumes the Reference (like comma); it actually preserves it.
- [x] **Explicit overrides: `call`, `apply`, `bind`** — `Function.prototype.call/apply` supply `this` directly; `bind` returns a BoundFunction exotic object with fixed `[[BoundThis]]`; partial application
      📊 shaky → solid — nailed [[Call]] vs [[Construct]] separation on remediation
      🔧 Refinements: — Predicted `call` overrides `bind` (wrong: wrapper intercepts). Predicted `new BoundFoo()` uses `[[BoundThis]]` (wrong: `[[Construct]]` never reads it).
- [x] **Quiz: values, Reference & `this` basics** — covers chunks 1–4
      📊 solid
      🔧 Refinements: — Forgot sloppy-mode ToObject coercion on primitives passed to `call`/`apply` (answered `"number"` instead of `"object"`).
- [x] **Arrow functions** — no own `[[ThisValue]]`/`arguments`/`new.target`; `this` resolved via `[[OuterEnv]]` chain walk; not constructable; the `self = this` problem they solve
      📊 solid
      🔧 Refinements: — Predicted `bind` overrides arrow's `this` (wrong: OrdinaryCallBindThis skips regardless of what delivers the thisValue).
- [x] **Constructor calls (`new`)** — OrdinaryCreateFromConstructor, `this` = fresh object, `new.target`, return-value override, `[[IsConstructor]]` internal slot
      📊 solid
- [x] **Class & `this`** — class methods as non-constructable, field initializer `this`-binding, `super()` as `this`-provider in derived constructors, `this`-TDZ before `super()`
      📊 solid
      🔧 Refinements:
        - On a return-value override + arrow field combo, said "arrow keeps `this` so it survives": correct mechanism in general, but missed that `b.read` resolves on the *returned* object (O2), not on the original (O1) — the override discards the entire instance the arrow lived on, so extraction-via-`b.read` fails because the property doesn't exist on O2 at all. Arrow-survives-by-closure only applies when the arrow reference itself escapes the constructor (e.g. assigned to a variable before the override returns).
        - Said "synthesized constructor" loosely; tightened: `class Dog extends Animal {}` synthesizes `constructor(...args) { super(...args); }`, which then calls `super()` — two layers. The synthesized constructor is what makes "no constructor written" still trigger the full chain.
        - In method-chaining (`c.bump().bump()`), called the returned value "c2"; it's the same object `c` (since `bump` does `return this`).
- [ ] **Patterns & pitfalls** — method extraction, callbacks, event handlers, setTimeout, historical workarounds, modern solutions (arrow fields, bind-in-constructor, decorator proposals)
      🔧 Refinements (sub-part 1):
        - **`setTimeout(fn, 0)` predicted as TypeError** — host dispatcher (Node `Timeout`, browser `window`) supplies `this` itself, so the symptom is silent `NaN`/wrong-object reads, not a loud throw. ECMAScript's strict-mode "plain call → undefined" rule doesn't apply when a host API is the caller.
        - **`forEach(c.tick)` predicted as no-op** — synchronous dispatchers count as dispatchers. `forEach` (and `map`/`filter`/`find`/etc.) calls the callback with `thisValue = undefined` unless `thisArg` is passed. Mistook "I haven't called it yet" for "no one calls it."
        - **General lesson** — "later" was misleading shorthand. The real axis is *who does the `[[Call]]`*: your code, or a dispatcher. Any dispatcher (sync or async) overrides the bare-function default with its own contract.
      🔧 Refinements (sub-part 2):
        - **Predicted `this.name` would print "Button"** on a `dispatchEvent`-supplied `this = button`. Got `this = button` right (mechanism solid), but didn't follow through to "does `EventTarget` have a `name` property?" — it doesn't, so `button.name` reads `undefined`. The class field `Reporter.name = "rep"` lives on the *original Reporter instance*, which was discarded at extraction; it's not on the object the dispatcher chose. Right conclusion direction (silent-`undefined`), wrong specific value.
- [ ] **Quiz: arrows, `new`, class** — covers chunks 5–8
- [ ] **Final test**
