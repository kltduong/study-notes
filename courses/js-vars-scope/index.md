# JS Variables, Scope & Execution Context

The mental model tying this course together: every variable declaration is a binding that goes through up to three lifecycle stages (declaration → initialization → assignment), and _where_ and _when_ those stages fire is determined by the spec machinery underneath — Execution Contexts, Environment Records, and the scope chain. Once you see that machinery, hoisting, TDZ, and scope rules stop being arbitrary rules and become inevitable consequences.

## Reading order

Start with `variable-lifecycle.md` — it introduces the 3-stage model that everything else references. Then `execution-context.md` for the spec structures that make those stages concrete. Then `creation-execution.md` for the two-phase algorithm that fills those structures at runtime. `hoisting.md` reframes hoisting as a consequence of that algorithm — read after `creation-execution.md`. Subsequent notes build on all four.

## Subtopic map

- [variable-lifecycle.md](variable-lifecycle.md) — the 3-stage model (declaration → initialization → assignment), how `var`/`let`/`const` differ at each stage, why the TDZ window exists
- [execution-context.md](execution-context.md) — Execution Context structure, Realm Record, Environment Record type hierarchy, `[[OuterEnv]]` chain, LexicalEnvironment vs VariableEnvironment pointers, `[[VarNames]]`
- [creation-execution.md](creation-execution.md) — the two-phase algorithm (creation setup pass + execution runtime pass), per-keyword rule during creation, nested create→execute pairs as the call stack, function declaration vs `var = fn expr` asymmetry
- [hoisting.md](hoisting.md) — hoisting as an observable (not a mechanism), per-keyword reachability before the declarator, why the "code moves up" folk image fails for `let`/`const`/`class`, function-declaration precedence, `typeof` and TDZ
- [tdz.md](tdz.md) — TDZ as a time window (not a source region), the one-flag/one-check mechanism, "temporal" meaning runtime not lexical, less obvious TDZ spots (default params, self-reference, class), design rationale, what TDZ pays for (block-shadowing + creation phase)
- [scope-model.md](scope-model.md) — the four scope types (global, function, block, module) as ER instances, two-axis model (keyword × pointer behavior), global object attachment and why it's problematic, script vs module topology
- [lexical-scoping.md](lexical-scoping.md) — the call-time pointer-copy axiom (`newEC.[[OuterEnv]] ← function.[[Environment]]`) that makes JS lexical, `ResolveBinding` walk, shadowing as first-match consequence, closures as ER survival via GC reachability, contrast with dynamic scoping (Bash)
- [var-quirks.md](var-quirks.md) — `var` quirks (re-declaration as idempotent registration, implicit globals as failed `ResolveBinding`) and historical patterns (IIFE as scope primitive, `"use strict"`) all derived from one axiom: pre-ES6, the function was the only construct that creates a new scope
- [const-immut.md](const-immut.md) — `const` as a binding-level constraint (`mutable: false` flag on the binding record, gated inside `SetMutableBinding`), `Object.freeze` as the orthogonal value-level lock (descriptor `[[Writable]]: false`), the two-layer composition and why full immutability needs both, practical defaults (const-first, freeze selectively)

## Cross-cutting concepts

- **Spec-level vs JS-level** — structures like EC and ER are spec-level: formally defined shapes the spec's algorithms operate on, not JS objects you can touch. Engines implement equivalent behavior but may look nothing like the spec internally.
- **Pointer vs instance** — `LexicalEnvironment`/`VariableEnvironment` are pointers on the EC; `[[DeclarativeRecord]]`/`[[ObjectRecord]]` are fields holding ER instances. The type hierarchy describes shapes; the fields hold the actual instances.
- **Two-phase execution** — creation phase (bindings registered) then execution phase (code runs). The lifecycle stages map directly onto these phases. Covered in `variable-lifecycle.md`; the EC machinery behind them in `execution-context.md`.
