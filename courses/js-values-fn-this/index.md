# JS Values, Functions & `this`

How JavaScript determines `this` — derived from first principles rather than memorized case lists. The course builds from value semantics through the Reference type to the single rule that governs all normal-call `this` determination, then extends to explicit overrides, arrows, constructors, and classes.

## Reading order

Linear — each note builds on the previous:

1. [Values & memory model](values-memory.md) — what lives in a slot (primitives vs references), copy semantics, identity
2. [The Reference type](reference-type.md) — the spec-internal struct that preserves "where a value was accessed from" between expression evaluation and the call operator
3. [`this` determination](this-determ.md) — the one rule (Reference base → `[[ThisValue]]`), the three-term pipeline, strict/sloppy coercion, and the complete decision tree

## Cross-cutting concepts

- **Slot model.** Every binding is a fixed-size slot. This mental model carries from value storage through Reference Records to `[[ThisValue]]` — all are "a value in a slot."
- **GetValue as the boundary.** GetValue is the operation that extracts a value from a Reference, discarding the wrapper. It's the mechanism behind both property access and `this`-loss — any operation that needs the value (not the address) calls GetValue.
- **Per-call, not per-function.** `this` is stored in the Function ER (created fresh per call), not on the function object. The same function gets different `this` values from different call sites.
