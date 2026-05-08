# JS Functions & `this` — Course TOC

Progress tracker for the course.

## Calibration

**Starting point:** Knows EC/ER internals from js-vars-scope (Function ER, `[[ThisValue]]` slot, `[[OuterEnv]]` chain, creation/execution phases). Comfortable with function declarations, arrow functions, and `call`/`apply`/`bind` in daily use. Fuzzy on the spec-level mechanism — the Reference type, how the engine extracts `this` from a call expression, why certain patterns lose `this`.

**Goals:**

1. One-rule mental model for `this` determination (Reference base) + know exactly which mechanisms override it.
2. Understand functions as a call protocol — not just "run the body" but the full sequence from expression evaluation to EC creation.
3. Confidently predict `this` in any code without memorizing case lists.

**Planned arc:**

1. **The Reference type** — what the engine produces when evaluating member expressions; base + name as a structured intermediate value. The foundation everything else derives from.
2. **`this` determination** — the one rule (Reference base → `[[ThisValue]]`) and why plain calls yield `undefined`. Strict vs sloppy coercion.
3. **Explicit overrides: `call`, `apply`, `bind`** — mechanisms that bypass the Reference-base default. Partial application, the bound-function wrapper.
4. **Arrow functions** — structurally different (no `[[ThisValue]]`, no `arguments`, no `new.target`). Lexical `this` via `[[OuterEnv]]` chain walk. Why they exist (the `self = this` problem).
5. **Constructor calls (`new`)** — the object-creation protocol, `new.target`, return-value override rule, why `class` constructors are `new`-only.
6. **Class & `this`** — field initializers, method `this`-loss, `super()` and `this`-TDZ in derived constructors. Call-protocol aspects only (inheritance mechanics stay in js-inheritance).
7. **Patterns & pitfalls** — callbacks losing `this`, event handlers, method extraction, historical patterns (`self`/`that`), modern solutions (arrow fields, `bind` in constructor).

**Pacing notes:** Chunks 1–2 are the core — the Reference type is the new machinery; spend time there. Chunk 3 is moderate. Chunks 4–5 build on the foundation with clear derivations. Chunks 6–7 are application/synthesis. Probe "right conclusion, wrong mechanism" weakness — `this` is a prime area for coincidence-based reasoning.

**Dependencies:** js-vars-scope (EC, Function ER, `[[ThisValue]]` slot, `[[OuterEnv]]` chain). Feeds into js-inheritance (method dispatch, constructor `this`).

## Progress

- [ ] **The Reference type** — spec-level Reference Record, base/name/strict fields, how member expressions produce References, how plain identifiers produce References with ER base
- [ ] **`this` determination** — the single rule: extract base from Reference at call site → `[[ThisValue]]`; ER base → `undefined`; strict vs sloppy coercion to `globalThis`
- [ ] **Explicit overrides: `call`, `apply`, `bind`** — `Function.prototype.call/apply` supply `this` directly; `bind` returns a BoundFunction exotic object with fixed `[[BoundThis]]`; partial application
- [ ] **Quiz: Reference & `this` basics** — covers chunks 1–3
- [ ] **Arrow functions** — no own `[[ThisValue]]`/`arguments`/`new.target`; `this` resolved via `[[OuterEnv]]` chain walk; not constructable; the `self = this` problem they solve
- [ ] **Constructor calls (`new`)** — OrdinaryCreateFromConstructor, `this` = fresh object, `new.target`, return-value override, `[[IsConstructor]]` internal slot
- [ ] **Class & `this`** — class methods as non-constructable, field initializer `this`-binding, `super()` as `this`-provider in derived constructors, `this`-TDZ before `super()`
- [ ] **Quiz: arrows, `new`, class** — covers chunks 4–6
- [ ] **Patterns & pitfalls** — method extraction, callbacks, event handlers, setTimeout, historical workarounds, modern solutions (arrow fields, bind-in-constructor, decorator proposals)
- [ ] **Final test**
