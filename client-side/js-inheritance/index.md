# Prototypal Inheritance

JavaScript doesn't have classical inheritance — it has **prototype delegation**. Objects link to other objects via a hidden `[[Prototype]]` slot, and property access walks that chain until it finds what it's looking for or hits `null`. Everything in these notes builds on that single mechanic.

## Reading order

Start with **Foundations**, then **[[Prototype]] Deep Dive**, then **Chain & Behavior**, then **Instantiation Patterns**. Foundations establishes the chain-walk mental model; the deep dive zooms in on the link itself and the edges (primitives, auto-boxing, `this` binding); chain & behavior covers what happens during the walk — shadowing, enumeration, and mutation traps; instantiation patterns shows the five ways to wire that chain in practice and why each exists.

## Notes

- [Foundations](foundations.md) — Why prototypes exist, delegation vs copying, the chain-walk invariant, `Object.prototype` as common ancestor.
- [[[Prototype]] Deep Dive](prototype-deep-dive.md) — Internal slot vs property, `Object.keys`/`for...in` differences, primitive auto-boxing, built-in prototype fan, custom prototypes and `this` at the call site.
- [Chain & Behavior](chain-behavior.md) — Shadowing (reads walk up, writes stay put), mutable reference trap, chain termination at `null`, enumerability and iteration tools, why runtime prototype mutation kills performance.
- [Instantiation Patterns](instantiation.md) — Five patterns (functional → functional shared → prototypal → pseudoclassical → class), class fields vs prototype methods, `[[Prototype]]` as pointer to an object not a variable.

## Cross-cutting concepts

- **Chain walk** — the core mechanic. Every note assumes it. Property lookup follows `[[Prototype]]` links until found or `null`.
- **Reads walk up, writes stay put** — the two-rule model for property access vs assignment. Explains shadowing, per-instance state, and the mutable reference trap.
- **Delegation, not copying** — methods aren't duplicated onto objects; they're found at lookup time via the chain.
- **`this` is call-site-determined** — critical for shared methods on prototypes. What's left of the dot wins.
- **`[[Prototype]]` is a pointer to an object, not a variable** — reassigning `Constructor.prototype` doesn't update existing instances. Wiring happens at creation time.
