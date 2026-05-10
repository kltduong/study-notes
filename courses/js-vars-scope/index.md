# JS Variables, Scope & Execution Context

The mental model tying this course together: every variable declaration is a binding that goes through up to three lifecycle stages (declaration → initialization → assignment), and _where_ and _when_ those stages fire is determined by the spec machinery underneath — Execution Contexts, Environment Records, and the scope chain. Once you see that machinery, hoisting, TDZ, and scope rules stop being arbitrary rules and become inevitable consequences.

## Reading order

Start with `variable-lifecycle.md` — it introduces the 3-stage model that everything else references. Then `execution-context.md` for the spec structures that make those stages concrete. Then `creation-execution.md` for the two-phase algorithm that fills those structures at runtime. Subsequent notes build on all three.

## Subtopic map

- [variable-lifecycle.md](variable-lifecycle.md) — the 3-stage model (declaration → initialization → assignment), how `var`/`let`/`const` differ at each stage, why the TDZ window exists
- [execution-context.md](execution-context.md) — Execution Context structure, Realm Record, Environment Record type hierarchy, `[[OuterEnv]]` chain, LexicalEnvironment vs VariableEnvironment pointers, `[[VarNames]]`
- [creation-execution.md](creation-execution.md) — the two-phase algorithm (creation setup pass + execution runtime pass), per-keyword rule during creation, nested create→execute pairs as the call stack, function declaration vs `var = fn expr` asymmetry

## Cross-cutting concepts

- **Spec-level vs JS-level** — structures like EC and ER are spec-level: formally defined shapes the spec's algorithms operate on, not JS objects you can touch. Engines implement equivalent behavior but may look nothing like the spec internally.
- **Pointer vs instance** — `LexicalEnvironment`/`VariableEnvironment` are pointers on the EC; `[[DeclarativeRecord]]`/`[[ObjectRecord]]` are fields holding ER instances. The type hierarchy describes shapes; the fields hold the actual instances.
- **Two-phase execution** — creation phase (bindings registered) then execution phase (code runs). The lifecycle stages map directly onto these phases. Covered in `variable-lifecycle.md`; the EC machinery behind them in `execution-context.md`.
