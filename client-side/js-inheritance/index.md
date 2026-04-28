# Prototypal Inheritance

JavaScript doesn't have classical inheritance — it has **prototype delegation**. Objects link to other objects via a hidden `[[Prototype]]` slot, and property access walks that chain until it finds what it's looking for or hits `null`. Everything in these notes builds on that single mechanic.

## Reading order

Start with **Foundations**, then **[[Prototype]] Deep Dive**. Foundations establishes the chain-walk mental model; the deep dive zooms in on the link itself and the edges (primitives, auto-boxing, `this` binding).

## Notes

- [Foundations](foundations.md) — Why prototypes exist, delegation vs copying, the chain-walk invariant, `Object.prototype` as common ancestor.
- [[[Prototype]] Deep Dive](prototype-deep-dive.md) — Internal slot vs property, `Object.keys`/`for...in` differences, primitive auto-boxing, built-in prototype fan, custom prototypes and `this` at the call site.

## Cross-cutting concepts

- **Chain walk** — the core mechanic. Every note assumes it. Property lookup follows `[[Prototype]]` links until found or `null`.
- **Delegation, not copying** — methods aren't duplicated onto objects; they're found at lookup time via the chain.
- **`this` is call-site-determined** — critical for shared methods on prototypes. What's left of the dot wins.
