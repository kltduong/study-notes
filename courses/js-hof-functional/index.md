# Higher-Order Functions & Functional Patterns

Functional programming in JS isn't a separate paradigm bolted on — it's a consequence of functions being first-class objects. Two axioms (functions are objects + one value-passing rule) give you HOFs, closures-as-factories, composition, currying, and every pattern in this course. The complexity is in the *design* (which function to pass, what to close over, how to compose), not in the mechanism.

## Reading order

Start with `fns-as-values.md` — it establishes the two axioms everything else derives from. Then `iter-abstractions.md` — the three structural shapes (map/filter/reduce) that form the core vocabulary for the rest of the course. Then `reduce.md` — deep dive into the universal fold, since most non-trivial functional patterns route through it. Then `composition.md` — gluing functions end-to-end with `pipe`/`compose`, point-free style, method chaining vs free functions, and transducers for fusing passes. Then `currying-partial.md` — the bridge between multi-argument functions and unary composition.

## Subtopic map

- [fns-as-values.md](fns-as-values.md) — First-class functions, the two axioms (functions are objects + one value-passing rule), HOF structural split, factory pattern via closures, function identity and reference semantics.
- [iter-abstractions.md](iter-abstractions.md) — The three iteration shapes: map (1-to-1 transform), filter (subset selection), reduce (aggregation). Chaining as pipelines, filter-early principle, relationship to `for` loops.
- [reduce.md](reduce.md) — Reduce as the universal fold. Accumulator design (init = empty shape; identity element for type-uniform, empty container for type-changing). Universality + the early-exit caveat. Mutate-vs-immutable accumulator (O(n) vs O(n²)) and the *externally pure, internally mutable* principle. Common pitfalls including the strict-vs-sloppy mode interaction.
- [composition.md](composition.md) — Function composition (`pipe`/`compose`), point-free style and the arity trap, method chaining vs functional composition (closed-set vs open-set), transducers-lite (fusing map/filter without intermediate arrays).
- [currying-partial.md](currying-partial.md) — Partial application (fix some args, get a smaller function) vs currying (restructure into unary chain). `bind`, manual wrappers, auto-curry utility. Parameter order convention (config first, data last). Arity limits and variadic workarounds.

## Cross-cutting concepts

- **Two axioms** — functions are objects + one value-passing rule. Every pattern in the course derives from these.
- **Shape as communication** — named methods (`map`, `filter`, `reduce`) declare structural guarantees in the method name. Prefer them over loops for self-documenting intent.
- **References, not copies** — array methods create new containers but share element references with the original. Mutation through either reference is visible.
- **Closure as the bridge** — factories, currying, memoization all rely on returning a function that closes over configuration state.
- **Currying as composition bridge** — multi-arg functions can't enter `pipe`/`compose` directly. Currying restructures them into unary chains; each partial application produces a pipeline-ready step. Convention: config/stable args first, data last.
- **Composition as monoid** — functions under composition have identity (`id = x => x`) and associativity (`pipe(f, g, h)` groups freely). Same algebraic structure as string concat, array concat, `+`.
- **Closed-set vs open-set** — method chaining is limited to what the prototype provides; functional composition works with any unary function. The expression problem in miniature.
- **Transducers decouple shape from logic** — "what each step does" (map/filter) is independent of "what shape we're building" (array, sum, observable). Same pipeline, multiple output shapes.
