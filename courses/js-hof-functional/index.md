# Higher-Order Functions & Functional Patterns

Functional programming in JS isn't a separate paradigm bolted on — it's a consequence of functions being first-class objects. Two axioms (functions are objects + one value-passing rule) give you HOFs, closures-as-factories, composition, currying, and every pattern in this course. The complexity is in the *design* (which function to pass, what to close over, how to compose), not in the mechanism.

## Reading order

Start with `fns-as-values.md` — it establishes the two axioms everything else derives from. Then `iter-abstractions.md` — the three structural shapes (map/filter/reduce) that form the core vocabulary for the rest of the course.

## Subtopic map

- [fns-as-values.md](fns-as-values.md) — First-class functions, the two axioms (functions are objects + one value-passing rule), HOF structural split, factory pattern via closures, function identity and reference semantics.
- [iter-abstractions.md](iter-abstractions.md) — The three iteration shapes: map (1-to-1 transform), filter (subset selection), reduce (aggregation). Chaining as pipelines, filter-early principle, relationship to `for` loops.

## Cross-cutting concepts

- **Two axioms** — functions are objects + one value-passing rule. Every pattern in the course derives from these.
- **Shape as communication** — named methods (`map`, `filter`, `reduce`) declare structural guarantees in the method name. Prefer them over loops for self-documenting intent.
- **References, not copies** — array methods create new containers but share element references with the original. Mutation through either reference is visible.
- **Closure as the bridge** — factories, currying, memoization all rely on returning a function that closes over configuration state.
