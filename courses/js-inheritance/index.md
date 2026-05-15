# Prototypal Inheritance

JavaScript doesn't have classical inheritance — it has **prototype delegation**. Objects link to other objects via a hidden `[[Prototype]]` slot, and property access walks that chain until it finds what it's looking for or hits `null`. Everything in these notes builds on that single mechanic.

## Reading order

Start with **Foundations**, then **[[Prototype]] Deep Dive**, then **Chain & Behavior**, then **Instantiation Patterns**, then **`__proto__`**, then **Modern Access**, then **`.prototype` Property**, then **Building Chains**. Foundations establishes the chain-walk mental model; the deep dive zooms in on the link itself and the edges (primitives, auto-boxing, `this` binding); chain & behavior covers what happens during the walk — shadowing, enumeration, and mutation traps; instantiation patterns shows the five ways to wire single-level chains in practice and why each exists; `__proto__` explains the legacy accessor that made `[[Prototype]]` accessible before modern APIs, and when its magic breaks; modern access covers the official APIs (`getPrototypeOf`, `setPrototypeOf`, `Object.create`) that bypass property lookup entirely, and why wiring at creation beats mutating after the fact; `.prototype` property resolves the naming confusion — it's a regular property on functions, not the internal slot, and `new` is the bridge between the two; building chains scales the single-level picture to multi-level inheritance — the two-layer thesis (prototype layer + instance layer), the four wiring mechanisms over time, and what `class extends` actually bundles.

## Notes

- [Foundations](foundations.md) — Why prototypes exist, delegation vs copying, the chain-walk invariant, `Object.prototype` as common ancestor.
- [[[Prototype]] Deep Dive](prototype-deep-dive.md) — Internal slot vs property, `Object.keys`/`for...in` differences, primitive auto-boxing, built-in prototype fan, custom prototypes and `this` at the call site.
- [Chain & Behavior](chain-behavior.md) — Shadowing (reads walk up, writes stay put), mutable reference trap, chain termination at `null`, enumerability and iteration tools, why runtime prototype mutation kills performance.
- [Instantiation Patterns](instantiation.md) — Five patterns (functional → functional shared → prototypal → pseudoclassical → class), class fields vs prototype methods, `[[Prototype]]` as pointer to an object not a variable.
- [`__proto__`](dunder-proto.md) — The legacy accessor on `Object.prototype`, Annex B status, when the magic disappears (`Object.create(null)`, `defineProperty` shadowing), prototype pollution.
- [Modern Access](modern-access.md) — `Object.getPrototypeOf`, `Object.setPrototypeOf`, `Object.create`, `Reflect` variants. Why these bypass every `__proto__` failure mode, and why wiring at creation beats post-creation mutation.
- [`.prototype` Property](dot-prototype.md) — The regular property on functions (not the internal slot). Only functions with `[[Construct]]` have it. `new` reads it to wire `[[Prototype]]` for instances. `.constructor`, overwriting gotchas, and the full two-arrow picture.
- [Building Chains](building-chains.md) — Multi-level inheritance has two orthogonal layers (prototype + instance). Four wiring mechanisms over time: `new Parent()` → `Object.create` → `setPrototypeOf` → `class extends`. The constructor-side chain for static inheritance. Two-layer independence as a debugging tool.

## Cross-cutting concepts

- **Chain walk** — the core mechanic. Every note assumes it. Property lookup follows `[[Prototype]]` links until found or `null`.
- **Reads walk up, writes stay put** — the two-rule model for property access vs assignment. Explains shadowing, per-instance state, and the mutable reference trap.
- **Delegation, not copying** — methods aren't duplicated onto objects; they're found at lookup time via the chain.
- **`this` is call-site-determined** — critical for shared methods on prototypes. What's left of the dot wins.
- **`[[Prototype]]` is a pointer to an object, not a variable** — reassigning `Constructor.prototype` doesn't update existing instances. Wiring happens at creation time.
- **`.prototype` vs `[[Prototype]]`** — `.prototype` is a property on functions, for future instances. `[[Prototype]]` is an internal slot on every object, for the object itself. `new` bridges the two.
