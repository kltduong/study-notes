# Temporal Dead Zone (TDZ)

**TL;DR:** TDZ is the time window between a binding's creation (stage 1 — name registered in the ER) and its initialization (stage 2 — declarator line executes). During this window, any read throws `ReferenceError: Cannot access 'x' before initialization`. "Temporal" means the dead zone is a runtime property — it depends on _when_ a read executes, not _where_ it sits in source. TDZ is the cheapest mechanism for making the stage-1/stage-2 gap safe, and it's the unavoidable tax on having both block-scoped shadowing and a creation phase.

## The mechanism — one flag, one check

From [variable-lifecycle.md](variable-lifecycle.md) and [creation-execution.md](creation-execution.md):

1. **Creation phase** registers `let`/`const`/`class` bindings with an **uninitialized** marker — the binding exists but holds no value (not even `undefined`).
2. **Every read** on a binding checks: uninitialized? → `ReferenceError`. Otherwise → return the value.
3. **The declarator line**, when execution reaches it, evaluates the initializer and flips the binding to initialized. Stage 2 = this flip.

TDZ is not a separate feature. It's the natural consequence of:
- stage 1 happening eagerly in creation (binding exists),
- stage 2 being deferred to the declarator line (binding is uninitialized until then),
- the engine throwing on reads of uninitialized bindings.

The "zone" is wall-clock time between scope entry and the declarator's execution. No bytecode pass marks off TDZ regions — there's just a flag and a check.

## "Temporal" — runtime, not lexical

Two flavors of `ReferenceError` illustrate the axis:

**Lexical — position in source decides the outcome:**

```js
function f() {
  let x = 1;
}
f();
console.log(x); // ReferenceError: x is not defined
```

No name resolution path reaches `x`. Timing is irrelevant — the read's _position in source_ decides everything.

**TDZ — time of execution decides the outcome:**

```js
const getX = () => x; // L1 — one read, one source position
getX();               // L2 — ReferenceError
let x = 10;           // L3 — stage 2 fires
getX();               // L4 — returns 10
```

L1's read doesn't move. Same scope, same binding, same name resolution path. Only the wall-clock moment of each call differs — and that flips the outcome. A lexical error couldn't do that: if L1 were "not in scope," every call would fail regardless of timing.

The engine distinguishes the two in the error message:
- Lexical: `ReferenceError: x is not defined`
- TDZ: `ReferenceError: Cannot access 'x' before initialization`

This is why it's called **Temporal** Dead Zone — the dead zone is a _time window_, not a _region of source code_. The window aligns with source lines in straight-line code, which is why the "region above the `let`" folk image works most of the time — but the underlying rule is temporal.

## The "above / below" folk rule breaks

_"You can't read a `let` above its declaration — below is fine."_ Works for flat code. Breaks when functions defer execution:

```js
function outer() {
  inner();           // L2 — inner's body runs NOW, before L3
  let x = 1;         // L3
  function inner() {
    console.log(x);  // L5 — textually below L3, but executes before it
  }
}
outer(); // ReferenceError at L5
```

L5 is textually below the declarator, but `inner()` is called at L2 — so L5 executes _before_ L3 flips `x` to initialized.

**The rule that actually works:** a binding is out of TDZ iff its declarator line has executed in the current lifetime of its scope. Source position is only a proxy — good in straight-line code, bad once functions or control flow are involved.

## Block-scoped shadowing and TDZ

```js
let x = 1;
{
  console.log(x);  // ReferenceError — NOT 1
  let x = 2;
}
```

At block entry, the Block ER's creation phase registers the inner `x` (stage 1). **Shadowing happens at stage 1** — the outer `x` is immediately unreachable from within the block, even though the inner `x` holds no value yet. L3's name resolution finds the inner `x` first, hits the uninitialized flag, throws.

This is the design payoff: without TDZ, L3 would silently read `undefined` (or worse, the outer `x`), and you'd have a subtle bug. TDZ forces the contradiction — "the inner `x` shadows but has no value yet" — into a loud error.

## Less obvious TDZ spots

### Default parameters

Parameters bind left-to-right; each default expression runs with previous parameters initialized but later ones still in TDZ:

```js
function f(a = b, b = 1) { /* ... */ }
f();        // ReferenceError: Cannot access 'b' before initialization
f(5);       // fine — a = 5 (default skipped), b defaults to 1

function g(a = 1, b = a) { /* ... */ }
g();        // fine — a = 1 first, then b defaults to a (already initialized)
```

Mechanism: the parameter list has its own Declarative ER with per-parameter stage 2 — same rule as `let`, applied to each slot in order.

### Self-reference in the initializer

The RHS runs _before_ stage 2 flips the binding:

```js
let x = x;          // ReferenceError
const y = y + 1;    // ReferenceError
```

The LHS `x` is the binding being declared (stage 1 already fired — it shadows any outer `x`). The RHS `x` is a read of that same uninitialized binding. Throws.

Contrast with `var` (stages 1+2 collapsed in creation):

```js
var x = x;          // no error — x is undefined on the RHS, assigned undefined
```

### `class` declarations

Classes follow the `let` rule — declared but uninitialized during creation, stage 2 at the declarator:

```js
const dog = new Dog();  // ReferenceError
class Dog {}
```

Why `class` can't behave like `function` (initialize in creation): class bodies can contain user code — `extends someExpr()`, computed method names, static field initializers — that must evaluate to construct the class object. Running user code during the no-code-runs creation phase would violate the axiom. Function declarations sidestep this because their creation-phase work is just allocating a function object and packing the body code into internal fields — the body doesn't run until called.

### `typeof` is not a safety hatch on TDZ names

`typeof undeclared` is safe (returns `"undefined"`). `typeof inTDZ` throws. The escape hatch only applies when no binding exists in the scope chain:

```js
typeof neverDeclared; // "undefined"
typeof x;             // ReferenceError
let x;
```

Declaring a `let` can make a name _less_ safe to probe than leaving it undeclared — counterintuitive, but consistent: any read on an uninitialized binding throws, `typeof` included.

## Design rationale — why throw?

Given that the stage-1/stage-2 window is unavoidable (see below), what should a read inside it do?

| Option                                 | Behavior        | Why rejected/chosen                                                                                                                                    |
| -------------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Auto-init to `undefined` (`var`-style) | returns `undefined` | Rejected. `let` would differ from `var` only in scoping — the bug class TDZ catches (use-before-initialize) stays silent.                          |
| New sentinel value readable from JS    | returns `uninitialized` | Rejected. Leaks a new primitive for one case. Still doesn't force the error — you'd have to remember to check.                                |
| Mark uninitialized, throw on read      | `ReferenceError` | **Chosen.** One flag per binding, one check per read. Catches the bug at the earliest observable point. Error message tells you exactly what's wrong. |

## What TDZ is really paying for

TDZ is the tax on having **both** of these at once:

- **Block-scoped shadowing** — the inner `let x` must win over any outer `x` from the `{` onward, or block scoping is broken.
- **A creation phase** — stage 1 must fire before any statement in the block runs, so the engine can static-scope-resolve names before execution.

Once both are in place, there _must_ be a wall-clock window where the inner binding exists but has no value — stage 1 fires at scope entry, stage 2 fires at the declarator, and scope entry precedes the declarator by construction.

Python avoids TDZ by not having a creation phase — `x = 1` is a single event, and references before it raise `NameError`. That also means Python can't support `var`-style hoisting, and block-scoped shadowing isn't a concern (no block scope). JS needed both features → needed the window → needed TDZ.

## Compressed takeaway

- **TDZ is a time window, not a source region.** Scope entry → declarator execution. "Above the `let`" only approximates this in straight-line code.
- **The mechanism:** stage 1 marks uninitialized; every read checks the flag; stage 2 flips it. One flag, one check.
- **Shadowing happens at stage 1.** The outer binding is unreachable from the new scope immediately — even before stage 2. TDZ errors are the runtime consequence.
- **Less obvious spots:** default params (per-parameter stage 2), self-reference (`let x = x`), classes (follow `let` rule), `typeof` (unsafe on TDZ names).
- **Design rationale:** throw vs `undefined` was a real choice. TDZ is the cheapest mechanism for making the window safe.
- **What it pays for:** block-scoped shadowing + creation phase → unavoidable window → TDZ.
