# Higher-Order Functions & Functional Patterns — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

_(To be filled at session start.)_

## Progress

- [ ] **Functions as values** — first-class functions, passing/returning functions, function references vs calls, closures as the enabling mechanism (bridge from js-values-fn-this)
- [ ] **Core iteration abstractions** — `map`, `filter`, `reduce` as folds over collections, how they compose, when each is the right tool, relationship to `for` loops
- [ ] **Reduce deep dive** — reduce as the universal fold, accumulator design, building other abstractions from reduce, common pitfalls (mutating accumulator, missing initial value)
- [ ] **Quiz 1** (chunks 1–3): first-class functions, iteration abstractions, reduce mechanics
- [ ] **Composition & pipelines** — function composition (`f(g(x))`), pipe/compose utilities, point-free style, when pipelines clarify vs obscure, transducers-lite (compose map/filter without intermediate arrays)
- [ ] **Currying & partial application** — currying vs partial application (distinct concepts), manual currying, `bind` as partial application, use cases (config factories, event handlers), arity and variadic functions
- [ ] **Quiz 2** (chunks 4–5): composition, currying, partial application
- [ ] **Immutability patterns** — pure functions, avoiding side effects, shallow copy idioms (spread, `Object.assign`), structural sharing concept, when immutability helps (predictability, debugging) vs when it hurts (performance, ergonomics)
- [ ] **Algebraic structure** — monoid pattern (identity + associative binary op), recognizing monoids in JS (string concat, array concat, `&&`, `||`, `+`), why `reduce` needs an initial value (identity element), functor-lite (mappable containers)
- [ ] **Quiz 3** (chunks 6–7): immutability, algebraic patterns
- [ ] **Test 1** (part 1: chunks 1–7): full functional patterns coverage
- [ ] **Real-world patterns** — method chaining vs pipeline, lazy evaluation (generators as lazy sequences), memoization, debounce/throttle as higher-order functions, decorator pattern
- [ ] **FP vs OOP tradeoffs** — when functional style wins (data transformation, stateless logic), when OOP wins (stateful entities, polymorphic dispatch), mixing styles idiomatically in JS, recognizing over-abstraction
- [ ] **Quiz 4** (chunks 8–9): real-world patterns, tradeoff reasoning
- [ ] **Final test** (cumulative)
