# Higher-Order Functions & Functional Patterns — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

**Starting point:** Uses HOFs daily (map/filter/reduce, callbacks, arrow functions). Closures, first-class functions, `this`/arrows/bind all solid from js-vars-scope and js-values-fn-this. Hasn't formalized the underlying structure — folds, monoids, composition laws, algebraic patterns.

**Goals:**

1. Compact axiom set for functional patterns in JS — derive behavior from structure, not memorize recipes.
2. Formal layer surfaced where it exists (folds, monoids, functors, composition associativity).
3. Clear decision framework: when FP style wins, when it doesn't, how to mix idiomatically.

**Planned arc:**

1. Functions as values — bridge from js-values-fn-this (quick, mostly review + formalize)
2. Core iteration abstractions — map/filter/reduce as structure
3. Reduce deep dive — the universal fold
4. Composition & pipelines — compose/pipe, point-free, transducers-lite
5. Currying & partial application — distinct concepts, arity, use cases
6. Immutability patterns — pure functions, copy idioms, structural sharing
7. Algebraic structure — monoids, functors, why reduce needs identity
8. Real-world patterns — memoization, debounce/throttle, decorators, lazy eval
9. FP vs OOP tradeoffs — decision framework, mixing styles

**Pacing notes:** *Functions as values* is light review — move fast. *Core iteration abstractions* and *Reduce deep dive* are the core formalization. *Composition & pipelines* and *Currying & partial application* are mechanism chunks. *Algebraic structure* is where the formal-abstraction preference pays off. Probe "right conclusion, wrong mechanism" on reduce semantics and composition associativity.

**Dependencies:** js-vars-scope (closures, EC/ER), js-values-fn-this (first-class functions, arrows, bind, value semantics).

## Progress

- [x] **Functions as values** — first-class functions, passing/returning functions, function references vs calls, closures as the enabling mechanism (bridge from js-values-fn-this)
      📊 solid
- [x] **Core iteration abstractions** — `map`, `filter`, `reduce` as folds over collections, how they compose, when each is the right tool, relationship to `for` loops
      📊 solid
- [x] **Reduce deep dive** — reduce as the universal fold, accumulator design, building other abstractions from reduce, common pitfalls (mutating accumulator, missing initial value)
      📊 solid
      📊 solid → shaky (Quiz 1 — protocol mechanics + sloppy-mode chain)
      🔧 Refinements (sub-part *Common pitfalls* check):
        - In a snippet with both "forgot return" and "missing init," cited "missing init" as the cause of the throw. Whether that's right depends on mode: in **strict mode**, missing init throws on iter 1 (`Cannot create property 'b' on string 'a'`); in **sloppy mode**, missing init silently no-ops on iter 1 and forgot-return throws on iter 2 (`Cannot read properties of undefined`). Diagnostic principle: when multiple bugs co-exist, strict mode tends to surface the earliest one by converting silent no-ops into throws.
      🔧 Refinements (Quiz 1):
        - **Q3 — JS `+` coercion**: predicted `obj + num` would throw (Python / strongly-typed instinct). Actual: JS coerces the object via `valueOf → toString` to `"[object Object]"`, then `+` flips to string concat. `obj + num` never throws in JS. (Cross-cutting clarification candidate for `js-values-fn-this`.)
        - **Q4 — `acc` vs `x` in reduce protocol**: stated the final result was the last `x`. Actual: reduce returns the final `acc`; the callback's return value replaces `acc`, `x` is the next array element — independent.
        - **Q5 — read vs write on primitives**: described the strict-mode throw as "engine tries to access `"a"["b"]`". Reads from primitives auto-box silently and return `undefined`; only *writes* to primitives are what strict mode promotes to TypeError. Strict throws on iter 1's `acc["a"] = 1` (write to string), not on a read.
        - **Q5 — sloppy-mode chain**: predicted "sloppy: no error, returns `"a"`". Sloppy mode silently swallows iter 1's write-to-primitive, then iter 2 throws on `undefined["a"]` because forgot-return left `acc = undefined`. Both modes throw — they differ on *which iteration* and *which underlying bug surfaces*.
- [x] **Quiz 1** (covering *Functions as values*, *Core iteration abstractions*, *Reduce deep dive*): first-class functions, iteration abstractions, reduce mechanics
      📊 fns-as-values: solid · iter-abstractions: solid · reduce: solid → shaky (protocol & sloppy-mode mechanics)
- [ ] **Composition & pipelines** — function composition (`f(g(x))`), pipe/compose utilities (built on `reduceRight` / `reduce` respectively), point-free style, when pipelines clarify vs obscure, transducers-lite (compose map/filter without intermediate arrays)
- [ ] **Currying & partial application** — currying vs partial application (distinct concepts), manual currying, `bind` as partial application, use cases (config factories, event handlers), arity and variadic functions
- [ ] **Quiz 2** (covering *Composition & pipelines*, *Currying & partial application*): composition, currying, partial application
- [ ] **Immutability patterns** — pure functions, avoiding side effects, shallow copy idioms (spread, `Object.assign`), structural sharing concept, when immutability helps (predictability, debugging) vs when it hurts (performance, ergonomics)
- [ ] **Algebraic structure** — monoid pattern (identity + associative binary op), recognizing monoids in JS (string concat, array concat, `&&`, `||`, `+`), why `reduce` needs an initial value (identity element), functor-lite (mappable containers)
- [ ] **Quiz 3** (covering *Immutability patterns*, *Algebraic structure*): immutability, algebraic patterns
- [ ] **Test 1** (part 1: covering *Functions as values* through *Algebraic structure*): full functional patterns coverage
- [ ] **Real-world patterns** — method chaining vs pipeline, lazy evaluation (generators as lazy sequences), memoization, debounce/throttle as higher-order functions, decorator pattern
- [ ] **FP vs OOP tradeoffs** — when functional style wins (data transformation, stateless logic), when OOP wins (stateful entities, polymorphic dispatch), mixing styles idiomatically in JS, recognizing over-abstraction
- [ ] **Quiz 4** (covering *Real-world patterns*, *FP vs OOP tradeoffs*): real-world patterns, tradeoff reasoning
- [ ] **Final test** (cumulative)
