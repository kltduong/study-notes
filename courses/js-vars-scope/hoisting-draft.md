# Hoisting — Draft

## Section 1 — What "hoisting" means

**Hoisting** is the folk name for this observable phenomenon: _a declared name is reachable from the entire scope it belongs to — including lines above its declaration in source order._

```js
foo(); // L1 — works. `foo` is reachable here, before its line.
function foo() {} // L2 — the declaration line
```

The call on L1 works. Something about `foo` is available _before_ the line where it's textually declared. That observation is what the word "hoisting" names — nothing more.

The term comes from an informal mental image: it _looks_ as if the declaration got lifted ("hoisted") to the top of its scope. Useful as a memory aid, misleading as a mechanism.

### Different forms of hoisting = different degrees of reachability

All declarations hoist in the sense that their name is reachable before the declaration line. What differs is _what reading the name gives you_:

| Declaration               | Before the declaration line, the name is reachable as… |
| ------------------------- | ------------------------------------------------------ |
| `function foo() {}`       | the function object — you can call it                  |
| `var x`                   | `undefined` — name exists, value not set yet           |
| `let` / `const` / `class` | declared-but-uninitialized — reads throw (TDZ)         |

Even the TDZ case counts as hoisting: the name isn't just "missing." It's _present_ in its scope (shadows outer names, blocks re-declaration, triggers TDZ errors instead of "not defined" errors). The binding exists from the top of its scope; what varies is usability.

### The mechanism behind the observable

All three behaviors fall out of the creation phase from [creation-execution.md](creation-execution.md) — the engine walks the scope before executing line one, registers every declared name, and the per-keyword rule decides what value (if any) the name starts with. "Hoisting" is just the folk term for what that phase makes observable.

Three distinct ideas, often conflated:

- **The phenomenon** — names are reachable before their declaration line.
- **The mechanism** — the creation phase registers names ahead of time per the per-keyword rule.
- **The misleading folk image** — "the engine rewrites the source and moves declarations up."

The folk image predicts `var` outcomes correctly often enough that people never notice it's wrong. It fails everywhere else:

```js
console.log(a); // folk image: "pretend `let a = 1` moved up → a is 1"
// reality: ReferenceError (TDZ)
let a = 1;
```

Same prediction failure for `const`, `class`, and function _expressions_ (Section 2). Drop the "code moves" image — the creation-phase model explains everything including TDZ, which the folk image can't.

## Section 2 — Are all declarations hoisted?

Yes — in the sense that every declaration registers its name during the creation phase. Every keyword reaches stage 1 before execution starts. What varies is whether stage 2 also fires in creation, deferring the usable window:

| Declaration form        | Name exists after creation? | Usable before its line?                                             |
| ----------------------- | --------------------------- | ------------------------------------------------------------------- |
| `function foo() {}`     | yes                         | **yes** — full value                                                |
| `var x = expr`          | yes                         | yes, but holds `undefined` until `= expr` runs                      |
| `let x = expr`          | yes                         | **no** — TDZ                                                        |
| `const x = expr`        | yes                         | **no** — TDZ                                                        |
| `class C {}`            | yes                         | **no** — TDZ                                                        |
| `var f = function() {}` | yes (name via `var`)        | yes, but holds `undefined` — the function value waits for execution |
| `var f = () => {}`      | same as above               | same as above                                                       |

The last two are the trap: a _function expression_ assigned to a `var` is just a `var` declaration. The RHS is an expression that evaluates during execution, exactly like `var f = computeSomething()`.

Compare the two things that people call "a function named foo":

```js
// L1 — function DECLARATION: name + value both ready after creation
hoistedFn(); // works

function hoistedFn() {
  return 1;
}

// L2 — function EXPRESSION assigned to var: only the name is ready
notHoistedFn(); // TypeError: notHoistedFn is not a function

var notHoistedFn = function () {
  return 1;
};
```

At the moment of the call:

- `hoistedFn` — binding holds the function object.
- `notHoistedFn` — binding holds `undefined`. Calling `undefined()` is a `TypeError`, not a `ReferenceError` — the name _exists_, just doesn't hold a callable.

The error type is diagnostic: `ReferenceError` means "name doesn't exist or is in TDZ," `TypeError` means "name exists, but the value doesn't support this operation."

### `class` — why TDZ and not "hoisted with its value"?

Classes are declared-only during creation, like `let`:

```js
new Dog(); // ReferenceError — Dog in TDZ
class Dog {}
```

Why not treat `class Dog {}` the way `function Dog() {}` is treated, initializing the class value during creation? Because a class body can contain arbitrary expressions in:

- `extends` clause: `class Sub extends computeParent() {}` — computing the parent requires running code.
- Computed method names: `class C { [someName()]() {} }`.
- Static field initializers: `class C { static x = expensive() }`.

Evaluating any of those during the creation phase would violate the "no code runs in creation" axiom. So classes stay uninitialized until the declarator line executes — the same window as `let`/`const`, for the same reason.

> **Aside — function declarations sidestep this** because their only evaluation during creation is `CreateFunctionObject`, which is a spec-internal allocation that doesn't evaluate user code. A class body can contain user code; a function's body doesn't run until it's called.

## Section 3 — Function-declaration precedence and the one-scope rule

Two quirks worth nailing down.

### Function declarations beat `var` in creation

If a `var` and a function declaration have the same name in the same scope, the function wins during creation — its value is what the binding holds when execution starts:

```js
console.log(typeof f); // "function" — not "undefined"

var f = 1;
function f() {}
```

After creation:

- `f` (as function declaration) registered and initialized to the function object.
- `var f` sees the name already exists; since `var` is idempotent at declaration time, it's a no-op for stage 1, and the already-present function value is kept (no `undefined` overwrite).

Then execution hits `var f = 1`, which runs as an assignment — now `f === 1`.

The ordering inside the scope doesn't matter for creation — `var f = 1; function f() {}` and `function f() {}; var f = 1;` behave identically, because creation processes all declarations before any statement runs.

### Duplicate declarations in one scope

- `var x` + `var x` in the same scope → legal, single binding.
- `var x` + `let x` in the same scope → `SyntaxError` at parse time (engine checks _before_ any code runs).
- `let x` + `let x` in the same scope → `SyntaxError`.
- `function f` + `function f` in the same scope → last one wins (the later declaration overwrites during creation).

These are parse-time errors, not runtime ones. The engine rejects the whole script before the creation phase even runs. This matters because a single `SyntaxError` means _no_ code in the affected scope runs at all — not even statements before the duplicate.

## Section 4 — The spec algorithms (for the nerd who asks)

If you want the formal names:

- **Global Declaration Instantiation** — runs once per script/module, populates the Global ER from top-level declarations.
- **Function Declaration Instantiation** — runs on each function call, populates the Function ER from parameters, `var`s, and inner function declarations.
- **Block Declaration Instantiation** — runs on block entry, populates the block's Declarative ER with `let`/`const`/`class` from the block.

All three walk the relevant AST region, apply the per-keyword rule, and set the ER's bindings. They're the formal name for the "creation phase setup pass" from `creation-execution.md`.

## Compressed takeaway

- **Hoisting = observable effect of the creation phase.** Nothing moves; the engine just knows all names before line one runs.
- **What differs per keyword is initialization, not declaration.** Every name reaches stage 1 in creation. Whether stage 2 fires in creation (`var`, `function`) or waits for the declarator line (`let`, `const`, `class`) is what produces the TDZ gap.
- **Function declarations ≠ `var f = function(){}`.** Declaration hoists name + value; the `var` form hoists only the name, leaving `undefined` until the assignment runs.
- **Function declarations beat `var` of the same name during creation**; `let`/`const` of the same name as anything is a `SyntaxError`.
- **Classes are TDZ, not initialized** — because class bodies can contain arbitrary expressions that can't run during creation.
- **`typeof` is not safe on TDZ names** — the only case where `typeof` can throw.
