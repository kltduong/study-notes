# TDZ — Draft

## Opening teaser

```js
const getX = () => x; // L1
getX();               // L2 — ReferenceError
let x = 10;           // L3
getX();               // L4 — 10
```

Same function. Same closure. Same binding. Only **time** changed.

- At L2: creation phase already ran, so `x` exists in the enclosing ER — but stage 2 hasn't fired. The binding is marked uninitialized. Reading it (from anywhere, including through a closure) throws.
- At L3: `let x = 10` runs. Stage 2 fires. TDZ ends for `x`.
- At L4: same read, same binding — now succeeds.

The key: **TDZ is a runtime property of a binding, not a lexical property of a source position.** Most reference errors you hit are lexical — "name not in scope here" is a static fact:

```js
function f() {
  let x = 1;
}
f();
console.log(x); // ReferenceError: x is not defined
```

This is a _lexical_ error — `x` was declared in `f`'s scope and doesn't exist in the outer scope. No name resolution path reaches it. It will throw no matter what ran before — there's no program state that makes `x` visible here. Determined entirely by where the read sits in the source.

Compare with TDZ — to really see the "runtime, not lexical" point, pin the read to a _fixed_ source position and vary only the time it executes. The opening teaser does exactly that:

```js
const getX = () => x; // L1 — one read, one source position
getX();               // L2 — throws
let x = 10;           // L3
getX();               // L4 — returns 10
```

L1's `return x` never moves. Same function, same binding, same name resolution path. Only the wall-clock moment differs between the two calls, and that flips the outcome. A purely lexical error couldn't do that — if the read at L1 were "not in scope," it would fail every single call, regardless of timing.

Both kinds throw `ReferenceError`, but the error messages differ:
- Lexical: `ReferenceError: x is not defined`
- TDZ: `ReferenceError: Cannot access 'x' before initialization`

TDZ breaks the usual pattern. The same line can succeed or throw depending on what ran before it.

This is why it's called **Temporal** Dead Zone and not, say, "spatial dead zone above the declarator." The dead zone is a _time window_, not a _region of source code_. The window happens to align with source lines in the simple cases (`console.log(x); let x = 1;`) — which is why the "region above the `let`" mental image works most of the time — but the underlying rule is temporal.

## The mechanism — one check, one flag

Pulling together from [variable-lifecycle.md](variable-lifecycle.md) and [creation-execution.md](creation-execution.md):

1. **Creation phase** registers `let`/`const`/`class` bindings in the appropriate ER with an **uninitialized** marker (spec: the binding exists but has no value — not `undefined`, not anything).
2. **Every read operation** on a binding checks: is this uninitialized? If yes → `ReferenceError`. If no → return the value.
3. **The declarator line**, when execution reaches it, evaluates the initializer (if any) and flips the binding to initialized, setting the value. Stage 2 = this flip.

That's it. TDZ is not a separate feature — it's the natural consequence of:
- stage 1 happening eagerly in creation (so the binding exists),
- stage 2 being deferred to the declarator line (so the binding is uninitialized until then),
- the engine throwing on reads of uninitialized bindings (instead of returning `undefined` like `var`).

The "zone" is the wall-clock time between scope entry and the declarator line's execution. No bytecode pass marks off TDZ regions in source — there's just a flag and a check.


## The "above / below" folk rule breaks

A natural folk rule people carry: _"you can't read a `let` above its declaration — below is fine."_ It works for flat straight-line code. It breaks the moment a function enters the picture, because functions defer _when_ their body runs.

```js
function outer() {
  inner();          // L2 — inner's body runs NOW, before L3
  let x = 1;        // L3
  function inner() {
    console.log(x); // L5 — textually below L3, but runs before it
  }
}
outer();
```

L5 is textually below the declarator, but `inner()` is called at L2 — so L5 executes _before_ L3 flips `x` to initialized. `ReferenceError`.

Flip it around: `let x = 1` _above_ a read doesn't save you either, if the declarator line never executes:

```js
function outer() {
  if (false) {
    let x = 1;      // never runs → stage 2 never fires
  }
  // x is not in scope here anyway — different issue (block-scoping),
  // but the general principle holds: "above" in source ≠ "already ran"
}
```

**The rule that actually works:** a binding is out of TDZ iff its declarator line has executed at least once in the current lifetime of its scope. Source position is only a proxy — a good one in straight-line code, a bad one once functions or control flow are involved.

## Less obvious TDZ spots

### Default parameters

Parameters are bound left-to-right, one at a time, and each parameter's default expression runs with the previous parameters already bound but later ones still in TDZ:

```js
function f(a = b, b = 1) { /* ... */ }
f();        // ReferenceError: Cannot access 'b' before initialization
f(5);       // fine — a = 5, b defaults to 1

function g(a = 1, b = a) { /* ... */ }
g();        // fine — a = 1 first, then b defaults to a
```

Mechanism: the parameter list has its own scope (a Declarative ER), and each parameter is registered with the same per-keyword rule as `let` — stage 1 eager, stage 2 at the parameter's own "declarator" (its slot in the list). At the moment `a`'s default `= b` evaluates, `b`'s stage 2 hasn't happened yet.

### Self-reference in the initializer

The declarator's right-hand side runs _before_ stage 2 flips the binding:

```js
let x = x;          // ReferenceError
const y = y + 1;    // ReferenceError
```

The LHS `x` is the one being declared in this scope (stage 1 already fired). The RHS `x` is a read of that same uninitialized binding — throws. This is a case where JS's block-scoped shadowing is load-bearing: there might be an outer `x`, but stage 1 of the inner `x` has already shadowed it by the time the RHS runs.

Contrast with `var`, where stages 1+2 collapsed in creation:

```js
var x = x;          // no error — x is undefined on the RHS, then assigned undefined
```

### `class` declarations

Classes follow the `let` rule (declared but uninitialized during creation, stage 2 at the declarator line). Same TDZ behavior:

```js
const dog = new Dog();  // ReferenceError
class Dog {}
```

Why `class` can't behave like `function` (initialize in creation) is covered in [hoisting.md](hoisting.md) — class bodies can contain user code (`extends someExpr()`, computed method names, static field initializers) that can't run during the no-code-runs creation phase.

### `typeof` is not a safety hatch on TDZ names

Touched on in [hoisting.md](hoisting.md), worth repeating here because it's a TDZ-specific surprise. `typeof undeclared` is safe — returns `"undefined"`. `typeof onTDZ` throws. The moment a name exists in the scope, `typeof` loses its escape-hatch property.

```js
typeof neverDeclared; // "undefined"
typeof x;             // ReferenceError
let x;
```

Declaring a `let` can make a name _less_ safe to probe than leaving it undeclared — counterintuitive, but consistent with the rule "any read on an uninitialized binding throws."


## Why throw instead of `undefined`?

The main design lever in this chunk is already in [variable-lifecycle.md](variable-lifecycle.md) — "static scoping forces a creation phase; deferring stage 2 opens a window; TDZ is how the window is made safe." But there's a narrower choice to call out: given that a window exists, what should a read inside it _do_?

Three options TC39 had:

| Option                                 | Behavior on read        | Why rejected/chosen                                                                                                                                                                           |
| -------------------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Auto-init to `undefined` (`var`-style) | returns `undefined`     | Rejected. Makes `let` differ from `var` only in scoping. The bug class TDZ catches (use-before-initialize) stays silent — the main safety benefit of separating stages 1 and 2 is lost.        |
| New sentinel value readable from JS    | returns `uninitialized` | Rejected. Leaks a new primitive into the language for one case. Still doesn't force the error — you'd have to remember to check for the sentinel.                                              |
| Mark uninitialized, throw on read      | `ReferenceError`        | **Chosen.** One flag per binding, one check per read. Catches the bug at the earliest observable point. The error message — `Cannot access 'x' before initialization` — tells you exactly what's wrong. |

## What TDZ is really paying for

TDZ is the tax on having **both** of these at once:

- **Block-scoped shadowing** — `{ let x; }` inside `let x = 1` must produce a binding that shadows the outer one from the `{` onward, or block scoping as a feature is broken.
- **A creation phase** — stage 1 must fire before any statement in the block runs, so that the engine can static-scope-resolve names (knows the inner `x` wins) before execution.

Once both are in place, there _must_ be a wall-clock window where the inner binding exists but has no value — because stage 1 fires at scope entry, stage 2 fires at the declarator, and scope entry precedes the declarator by construction.

Given that window is unavoidable, the remaining question is what to do with reads inside it — and that's the throw-vs-undefined choice above.

Python avoids TDZ by not having a creation phase at all — `x = 1` is a single event, and references before it raise `NameError` ("name 'x' is not defined"). That also means Python can't support `var`-style hoisting, and block-scoped shadowing isn't something Python has to worry about (no block scope). JS needed _both_ features, which means JS needed the window, which means JS needed TDZ.

## Compressed takeaway

- **TDZ is a time window, not a source region.** A binding is in TDZ from scope entry until its declarator line executes. "Above the `let`" only approximates this in straight-line code.
- **The mechanism:** stage 1 marks the binding uninitialized; every read checks the flag; stage 2 flips it. One flag, one check.
- **Shadowing happens at stage 1.** As soon as creation registers the inner `let`, the outer same-name binding is unreachable from the new scope — even before stage 2 runs. TDZ errors are the runtime consequence of that decision.
- **Defaults, self-reference, classes** — anywhere a read can happen before the declarator's stage 2, TDZ applies. Default params have per-parameter stage 2; `let x = x` reads the shadowing binding; classes follow the `let` rule.
- **`typeof` is unsafe on TDZ names.** Any read throws, including `typeof`. The undeclared-safe escape hatch only applies when no binding exists.
- **Design rationale:** throw vs `undefined` was a real choice. TDZ is the cheapest mechanism for making the stage-1/stage-2 window safe, and the one that gives the clearest error.
