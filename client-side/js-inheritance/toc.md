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

- [x] **Foundations** — What JS is, why objects and prototypes matter, everything inherits from Object (Section 1: intro through "Use prototypes when creating objects with a similar type")
- [x] **[[Prototype]] deep dive** — The hidden `[[Prototype]]` property, getting/setting prototypes, primitives vs objects, object wrappers, creating custom prototypes, prototype of object literals, inherited methods (Section 2: "Why understanding prototypes matter" through "Setting prototypes example")
- [x] **Prototype chain & behavior** — Chain mechanics, null termination, shadowing, `this` binding, enumerable properties, iteration, warnings about mutating prototypes (Section 2 continued: "The prototype chain" through "Summary")
- [ ] **[[Prototype]] quiz**
- [ ] **Instantiation patterns** — Functional, Functional Shared, Prototypal (`Object.create`), Pseudoclassical (`new` + constructor), Class syntax, arrow function caveats (Section 3: "Object Literals" through "Instantiation Patterns Quiz")
- [ ] **Prototypes & instantiation patterns test**
- [ ] **`__proto__`** — What it is, ECMAScript history, Annex B deprecation, pronunciation, defining custom getter/setter, exercises (Section 4: "**proto** introduction" through "**proto** quiz")
- [ ] **Modern prototype access** — `Object.getPrototypeOf`, `Object.setPrototypeOf`, problems with `__proto__` (configurable, special keyword, null prototype), MDN warnings (Section 5: "Modern alternatives..." through "Hang the Dunder on the wall")
- [ ] **`.prototype` property** — Only functions have it, exceptions, `[[Call]]`/`[[Construct]]`, constructor property, overwriting `.prototype`, full chain revealed, `Object.prototype` at the top, `.prototype` vs `[[Prototype]]` (Section 6: "Only functions have a .prototype property" through "Function prototype quiz")
- [ ] **Advanced prototypes test**
- [ ] **Building prototype chains** — `new` keyword approach (pre-2011), `Object.create` (ES5), `Object.setPrototypeOf` (ES6), classes + `extends`, 3-level chains with constructors and classes, `call()` for inheritance, duplication problems (Section 7: "Course project overview" through "You've beaten prototypes")
- [ ] **OOP & class vs prototype** — What OOP means, class-based vs prototype-based inheritance, Java comparison, classes vs prototypes tradeoffs (Section 8)
- [ ] **Composition** — Intro, converting prototypal model to composition, inheritance vs composition tradeoffs, when to use which (Section 9)
- [ ] **Outro & next steps**
