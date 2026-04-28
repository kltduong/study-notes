# Prototypal Inheritance — Course TOC

Progress tracker for the course. Chunks grouped by theme.

- [ ] **Foundations** — What JS is, why objects and prototypes matter, everything inherits from Object (Section 1: intro through "Use prototypes when creating objects with a similar type")
- [ ] **[[Prototype]] deep dive** — The hidden `[[Prototype]]` property, getting/setting prototypes, primitives vs objects, object wrappers, creating custom prototypes, prototype of object literals, inherited methods (Section 2: "Why understanding prototypes matter" through "Setting prototypes example")
- [ ] **Prototype chain & behavior** — Chain mechanics, null termination, shadowing, `this` binding, enumerable properties, iteration, warnings about mutating prototypes (Section 2 continued: "The prototype chain" through "Summary")
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
