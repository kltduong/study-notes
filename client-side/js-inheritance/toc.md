# Prototypal Inheritance — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

**Starting point:** Between basics and solid understanding — knows objects have prototypes and there's a chain, but the wiring details (`.prototype` vs `__proto__` vs `[[Prototype]]`, how `new` connects things) are fuzzy.

**Goals (priority order):**

1. Clean mental model — how it all fits together under the hood
2. Implementation fluency — confident writing and debugging prototype-based code
3. Deep mastery — edge cases, performance, when to use what

**Planned arc:**

1. **Foundations** — move quickly, confirm the mental model base, fill any gaps. Focus on _why_ prototypes exist, not definitions.
2. **[[Prototype]] deep dive** — this is where the core mental model gets built. Slow down here. Nail the hidden link, how lookup works, primitives vs objects.
3. **Prototype chain & behavior** — extend the model: chain traversal, shadowing, `this` binding. Concrete examples at every step.
4. **Instantiation patterns** — the "implementation fluency" chunk. Each pattern as a different way to wire the same chain. Write-it-yourself moments.
5. **`__proto__` → Modern access** — collapse these two chunks. History + deprecation + modern API in one pass. Practical: when you'd use each.
6. **`.prototype` property** — the trickiest naming confusion. Diagrams. Distinguish `.prototype` (function property) from `[[Prototype]]` (hidden link) once and for all.
7. **Building prototype chains** — multi-level chains, `extends`, `call()`. Hands-on construction.
8. **OOP & class vs prototype** — zoom out: mental model comparison, tradeoffs.
9. **Composition** — when inheritance is the wrong tool. Converting between patterns.
10. Quizzes/tests woven in at natural checkpoints, not as separate chunks.

## Progress

- [x] **Foundations** — What JS is, why objects and prototypes matter, everything inherits from Object (Section 1: intro through "Use prototypes when creating objects with a similar type")
- [x] **[[Prototype]] deep dive** — The hidden `[[Prototype]]` property, getting/setting prototypes, primitives vs objects, object wrappers, creating custom prototypes, prototype of object literals, inherited methods (Section 2: "Why understanding prototypes matter" through "Setting prototypes example")
- [x] **Prototype chain & behavior** — Chain mechanics, null termination, shadowing, `this` binding, enumerable properties, iteration, warnings about mutating prototypes (Section 2 continued: "The prototype chain" through "Summary")
- [x] **[[Prototype]] quiz**
- [x] **Instantiation patterns** — Functional, Functional Shared, Prototypal (`Object.create`), Pseudoclassical (`new` + constructor), Class syntax, arrow function caveats (Section 3: "Object Literals" through "Instantiation Patterns Quiz")
- [x] **Prototypes & instantiation patterns test**
- [x] **`__proto__`** — What it is, ECMAScript history, Annex B deprecation, pronunciation, defining custom getter/setter, exercises (Section 4: "**proto** introduction" through "**proto** quiz")
- [x] **Modern prototype access** — `Object.getPrototypeOf`, `Object.setPrototypeOf`, problems with `__proto__` (configurable, special keyword, null prototype), MDN warnings (Section 5: "Modern alternatives..." through "Hang the Dunder on the wall")
      📊 shaky — solid on setPrototypeOf perf cost and Object.create distinction; fuzzy on the specific cases where **proto** breaks (shadowing, null-prototype unreachability vs getter returning undefined)
- [x] **`.prototype` property** — Only functions have it, exceptions, `[[Call]]`/`[[Construct]]`, constructor property, overwriting `.prototype`, full chain revealed, `Object.prototype` at the top, `.prototype` vs `[[Prototype]]` (Section 6: "Only functions have a .prototype property" through "Function prototype quiz")
      📊 shaky — solid on .prototype vs [[Prototype]] distinction and [[Construct]]; missed that .constructor chain-walks to Object.prototype when overwritten
- [x] **Advanced prototypes quiz**
      📊 solid — core mechanics strong; chain walk, shadowing, class fields vs prototype methods all correct
      🔧 Refinements: - Q1: Correct conclusion on `Object.create(null)` + `__proto__` but reasoning skipped _why_ the accessor wasn't found — key point is the accessor lives on `Object.prototype` and the chain doesn't reach it - Q3: Confused property descriptor `{ value: 42 }` with the value itself (`42`) when `defineProperty` shadows `__proto__` - Q4: Said `bark` lives in "engine's internal slot" — it lives on `Dog.prototype`, a regular JS object; the _link_ is engine-internal, the _target_ is not
- [ ] **Building prototype chains** — `new` keyword approach (pre-2011), `Object.create` (ES5), `Object.setPrototypeOf` (ES6), classes + `extends`, 3-level chains with constructors and classes, `call()` for inheritance, duplication problems (Section 7: "Course project overview" through "You've beaten prototypes")
- [ ] **OOP & class vs prototype** — What OOP means, class-based vs prototype-based inheritance, Java comparison, classes vs prototypes tradeoffs (Section 8)
- [ ] **Composition** — Intro, converting prototypal model to composition, inheritance vs composition tradeoffs, when to use which (Section 9)
- [ ] **Final test**
- [ ] **Outro & next steps**
