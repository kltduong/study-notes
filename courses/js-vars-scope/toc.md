# JS Variables, Scope & Execution Context — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

**Starting point:** Comfortable with `var`/`let`/`const` in daily use, knows hoisting exists, understands execution contexts at a high level from async-js course (call stack, creation/execution phases). Fuzzy on the spec-level machinery — Realm Records, Environment Records, how TDZ is actually enforced, the formal scope chain resolution algorithm.

**Goals:**

1. Clean mental model of how the engine processes variable declarations — from parsing to runtime, at the spec level.
2. Implementation fluency — confidently predict scoping/hoisting/TDZ behavior in any code without guessing.
3. Connect execution context internals to the async-js mental model (same machinery, different angle).

**Planned arc:**

1. **Variable lifecycle** — start with the 3-stage model as the unifying framework. Quick but precise — this becomes the lens for everything after.
2. **Execution context internals** — slow down here. Realm Record, Environment Records, lexical vs variable environment. This is the spec machinery that makes hoisting/TDZ/scope _inevitable_ rather than arbitrary rules.
3. **Creation & execution phases** — walk through what actually happens step by step. Connect to the call stack model from async-js.
4. **Hoisting mechanics** — reframe hoisting as a _consequence_ of the creation phase, not a separate concept. Spec definitions of `var`/`let`/`const`.
5. **TDZ** — the design rationale, what "temporal" means formally, practical identification.
6. **Scope model** — all four scope types mapped to their Environment Record types. Global object attachment mechanism.
7. **Lexical scoping & shadowing** — scope chain resolution algorithm, shadowing rules, contrast with dynamic scoping.
8. **`var` quirks & historical patterns** — re-declaration, no block scope, IIFEs, strict mode. Historical context for surviving patterns.
9. **`const` & immutability** — reassignment vs mutation, `Object.freeze`, practical use cases.

**Pacing notes:** Chunks 2–3 are the core — spend time there. Chunks 1, 8, 9 are lighter (review + new angle). Probe descriptor-vs-value confusion and engine-vs-runtime conflation from cross-cutting weaknesses where relevant.

## Progress

- [ ] **Variable lifecycle** — 3 stages (declaration → initialization → assignment), how `var`/`let`/`const` differ at each stage, why JS separates declaration from initialization
- [ ] **Execution context internals** — EC types and phases, Realm Record (`[[Intrinsics]]`, `[[GlobalObject]]`, `[[GlobalEnv]]`), Environment Records, `[[VarNames]]`, lexical vs variable environment
- [ ] **Creation & execution phases** — what happens in each phase, variable/function setup, call stack, function execution contexts, worked examples
- [ ] **Hoisting mechanics** — hoisting as consequence of creation phase (not code movement), why `var` → `undefined`, `let`/`const` spec definitions per ECMAScript
- [ ] **TDZ** — design rationale, what "temporal" means, identifying TDZ in practice, relationship to the creation phase model
- [ ] **Quiz: execution model** — covers chunks 1–5
- [ ] **Scope model** — global / function / module / block scope, Environment Record types, `var` vs `let`/`const` scope rules, global object attachment and why it's problematic
- [ ] **Lexical scoping & shadowing** — scope chain resolution, nested scopes, shadowing rules, lexical vs dynamic scoping (with Bash contrast)
- [ ] **`var` quirks & historical patterns** — re-declaration, no block scope, IIFEs as pre-`let` workaround, `"use strict"`, when `var` is still appropriate
- [ ] **`const` & immutability** — reassignment vs mutation, `Object.freeze`, shallow vs deep, use cases
- [ ] **Quiz: scope & declarations** — covers chunks 6–9
- [ ] **Final test**
