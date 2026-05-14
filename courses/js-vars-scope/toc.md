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

- [x] **Variable lifecycle** — 3 stages (declaration → initialization → assignment), how `var`/`let`/`const` differ at each stage, why JS separates declaration from initialization
      📊 solid — core model strong; minor terminology slip (called shadowing "not shadowing" while describing the mechanism correctly)
      🔧 Refinements: - Called `let x = 5` "declaration, initialization, assignment" (3 stages) — it's only stages 1+2; the `= 5` is the initializer, not a stage-3 assignment - Described block-scoped shadowing as "without shadowing" while correctly explaining the shadowing mechanism
- [x] **Execution context internals** — EC types and phases, Realm Record (`[[Intrinsics]]`, `[[GlobalObject]]`, `[[GlobalEnv]]`), Environment Records, `[[VarNames]]`, lexical vs variable environment
      📊 solid — all correct; minor imprecision on Q2 (conflated EC pointer names with ER instances — draft ambiguity, fixed); "goes away" on Q3 correct in substance, precise mechanism is GC eligibility not instant deletion
      🔧 Refinements: - Said "LexicalEnvironment/VariableEnvironment are ERs" — they are pointers on the EC that point to ER instances; the draft diagram invited this, now fixed - Said `globalThis.x !== x` for the `let` case — more precisely, `globalThis.x` is `undefined` (property doesn't exist), not just a different value
- [x] **Creation & execution phases** — what happens in each phase, variable/function setup, call stack, function execution contexts, worked examples
      📊 solid — core model strong; per-keyword creation-phase rules, pointer routing (var→VarEnv, let→LexEnv), and fn-decl vs `var = fn-expr` asymmetry all landed
      🔧 Refinements: - Said `typeof function(){}` is `"object"` — it's `"function"` (the one arbitrary `typeof` result that isn't derivable, alongside `typeof null === "object"`) - Minor typo substituting `x` for `z` when describing the function-declaration creation step (reasoning was correct)
- [x] **Hoisting mechanics** — hoisting as consequence of creation phase (not code movement), why `var` → `undefined`, `let`/`const` spec definitions per ECMAScript
      📊 shaky — core model strong (observable vs mechanism split, per-keyword reachability, creation phase as mechanism); residual code-movement thinking on Q3 (treated `let x = 5` as equivalent to `let x;` + `x = 5;` — missed TDZ-window collapse and block-scope shift)
      🔧 Refinements: - Didn't know `let x;` (no initializer) fires stage 2 with `undefined` — assumed it stayed in TDZ. Folded into `variable-lifecycle.md` with a full 6-row declarator-form grid - Called `function() {}` on L4 of Q2 "the function is declared" — it's a function _expression_ that evaluates to a function value at runtime; only `var bar` is declared. Expression vs declaration - "Declares the value x" — we declare the binding/name, not the value; value is what the binding holds. Binding-vs-value precision - Asked "is `foo`'s code evaluated during creation?" — good clarifying question revealing the body-vs-object-allocation distinction. Folded the distinction into the answer: creation allocates the function object + packs body code into internal fields; body runs only when called
- [x] **TDZ** — design rationale, what "temporal" means, identifying TDZ in practice, relationship to the creation phase model
      📊 solid — temporal vs lexical distinction landed cleanly; default-param TDZ, block-shadowing TDZ, and the "above/below" folk-rule failure all correct; one EC-vs-ER terminology slip under pressure (block entry ≠ new EC)
      🔧 Refinements:
        - Said "inside the block, there is a new EC" — block entry creates a new Block ER and moves `LexicalEnvironment` to it; no new EC. ECs are only pushed on function calls (plus script/eval). Re-surfacing the EC-vs-ER distinction under pressure.
        - (Teaching-side correction) Q3 var-variant: teacher incorrectly said `var x` inside a block with outer `let x` would print `1` with "two separate bindings." Actual behavior: `SyntaxError` — `var` hoists to the same scope as the `let`, triggering the duplicate-declaration rule. Learner's instinct ("same binding") was pointing at the collision correctly.
- [x] **Quiz: execution model** — covers chunks 1–5
      📊 solid — all five correct; lifecycle stages, pointer mechanics, chain resolution, TDZ temporal behavior, Global ER routing all landed. One refinement: initially described `var` resolution as "via VariableEnvironment" rather than chain-walk from LexicalEnvironment — corrected after discussion (VarEnv is creation-time routing, not runtime lookup entry point)
- [x] **Scope model** — global / function / module / block scope, Environment Record types, `var` vs `let`/`const` scope rules, global object attachment and why it's problematic
      📊 solid — core derivation clean (two-axis model, pointer-pin vs pointer-move → scope rules fall out); module isolation landed; Q3 structural explanation precise
      🔧 Refinements:
        - Q1 initial prediction was `[30, 31, 32]` — double error: (a) assumed `i` reached `3` at loop exit (it stops at `2`), (b) imagined `x` still updating during L7 rather than frozen at `20` after loop exit. Corrected to `[20, 21, 22]` after tracing the chain.
        - Conflated the function **object**'s `[[Environment]]` slot (set once at creation, L5) with the running EC's `[[OuterEnv]]` (copied from `[[Environment]]` when the function is called on L7). Same ER referenced at two lifecycle points — creation-time capture vs call-time chain. Resurfacing the "closures capture a reference, not a value" mechanism at spec-level precision.
        - Said "var variable as 1 inside the whole function" when describing function scope — intent clear (single binding for the whole function) but wording slipped between "one" and "as 1".
        - Left implicit that `let` binding unreachability after block exit is caused by `LexicalEnvironment` **reverting** (Block ER becomes unreachable → GC-eligible), not by the block itself "ending". Pointer revert is the mechanism; block scope is the observable consequence.
- [x] **Lexical scoping & shadowing** — scope chain resolution, nested scopes, shadowing rules, closures (ER survival via `[[OuterEnv]]`), lexical vs dynamic scoping (with Bash contrast)
      📊 solid — capture/call-time bridge clean, scope chain walk + shadowing landed, closures derived from lexical-capture + GC reachability, lexical-vs-dynamic synthesis ("encapsulation") reached after one narrowing. Mechanism understanding strong throughout.
      🔧 Refinements:
        - Q1: said "a.[[Environment]] points to build Function ER when calling a()" — slot is actually set at function-object *allocation* (during the surrounding call's execution), not at the inner function's call time. Same definition-time-vs-call-time compression as chunk 6's `[[Environment]]` refinement. **Recurring pattern.**
        - Q1: applied first-match-wins and the lexical-capture rule correctly but didn't name them explicitly ("the effected ER is where the internal function is defined" rather than "scope chain is built from `[[OuterEnv]]`, which traces back to definition site"). Worth naming the formal rule so the pattern transfers cleanly to other traces.
        - Q2: misread `x = -1` (full assignment) as `x -= 1` (decrement). Mechanism reasoning intact under their reading — parsing slip, not conceptual error. Question switched to `x -= 1` for cleaner illustration; under the corrected text their answers (`[-1, 9, 19]` for `let`, `[19, 19, 19]` for `var`) were fully correct.
        - Q3 Scenario 2: gave the literal output (`[10, 20, 30]`) without surfacing the deeper failure mode — under dynamic-JS, library internals can silently clash with user-code free-variable names (encapsulation break). The deeper risk needed prompting; "literally correct, missing the lesson" pattern.
- [x] **`var` quirks & historical patterns** — re-declaration, no block scope, IIFEs as pre-`let` workaround, `"use strict"`, when `var` is still appropriate
      📊 solid — all three end-of-chunk questions correct first try; IIFE two-channel framing landed after the chunk-opener probe; spontaneously raised the great clarifying question "does the let/var collision rule work line-by-line or whole-scope?" which surfaced the Early Errors set-based property — folded into the note
      🔧 Refinements:
        - Q1 (a/b): described `var` / `let` global routing as "backed by globalThis" / "not backed by globalThis" — directionally right but loose. Precise mechanism is sub-record routing within the *composite* Global ER: `var` → `[[ObjectRecord]]` (which is the part backed by `globalThis`); `let` → `[[DeclarativeRecord]]` (the hidden table, no backing object). "Backed by globalThis" is a property of `[[ObjectRecord]]` specifically, not of the Global ER as a whole.
        - Q1 (d): explained the miss (`var inside` lives in the IIFE's Function ER) but didn't surface the `typeof` exemption as the reason `"undefined"` is returned rather than the lookup throwing `ReferenceError`. Recurring touch — same tightening landed during the chunk-opener (c). Lock in: `typeof` is the one operator spec'd to swallow `ReferenceError` for an undeclared name.
        - Chunk opener (d) / parse-trick: right conclusion ("syntax doesn't work") with mechanism imprecise. Precise framing: `function` at statement position parses as `FunctionDeclaration`, whose grammar requires a `BindingIdentifier`. Parens push the parse into expression position where `FunctionExpression` allows the name to be omitted. Recurring **right-conclusion / imprecise-mechanism** pattern from chunks 5 and 6.
        - Absorption check, predict (c) for strict mode duplicate params: initially said "fine" — recovered immediately when nudged at the strict-mode table. Forgot to apply a row already in the draft, not a mechanism gap.
- [ ] **`const` & immutability** — reassignment vs mutation, `Object.freeze`, shallow vs deep, use cases
- [ ] **Quiz: scope & declarations** — covers chunks 6–9
- [ ] **Final test**
