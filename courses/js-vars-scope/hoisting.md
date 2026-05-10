# Hoisting

**TL;DR:** Hoisting is the folk name for one observable: a declared name is reachable from the top of its scope, even on lines above the declaration. The mechanism is the creation phase from [creation-execution.md](creation-execution.md) — a setup pass that registers every declared name before any statement runs. All declarations hoist; what differs is the _initial value_ the binding holds during the window between scope entry and the declarator line (`function` → the function object; `var` → `undefined`; `let`/`const`/`class` → uninitialized, reads throw TDZ). The "engine moves declarations up" folk image predicts `var` outcomes often enough to go unnoticed, and fails everywhere else.

## What "hoisting" means

**Hoisting** is the folk term for this observable: _a declared name is reachable from the entire scope it belongs to — including lines above its declaration in source order._

```js
foo(); // L1 — works. `foo` is reachable here, above its line.
function foo() {} // L2
```

The term comes from the mental image of declarations being lifted ("hoisted") to the top of their scope. Useful as a memory aid, misleading as a mechanism. Nothing in the source text moves. Three distinct ideas to keep apart:

- **Phenomenon** — names are reachable before their declaration line.
- **Mechanism** — the creation phase registers names ahead of time per the per-keyword rule (see [creation-execution.md](creation-execution.md)).
- **Folk image** — "the engine rewrites the source and moves declarations up." It predicts `var` correctly and fails everywhere else.

## All declarations hoist; they differ in usability

Every declaration reaches stage 1 (name registered in the ER) during the creation phase. What varies is _what reading the name gives you_ before the declarator line runs:

| Declaration               | Before the declarator, the name is reachable as…               |
| ------------------------- | -------------------------------------------------------------- |
| `function foo() {}`       | the function object — callable                                 |
| `var x`                   | `undefined` — stage 2 already fired in creation                |
| `let` / `const` / `class` | declared-but-uninitialized — reads throw `ReferenceError` (TDZ) |

Even the TDZ case counts as hoisting: the name is **present** in its scope — it shadows outer names of the same identifier, blocks re-declaration, and throws a "before initialization" error instead of a "not defined" error. The binding exists from the top of the scope; what varies is whether it holds a usable value.

The lifecycle grid in [variable-lifecycle.md](variable-lifecycle.md) shows exactly what value each form holds at each phase.

## Where the folk image breaks

The "pretend declarations move to the top" mental shortcut fails on three counts:

### 1. It can't explain TDZ

```js
console.log(a); // folk image predicts: "pretend `let a = 1` moved up → a is 1"
// reality: ReferenceError
let a = 1;
```

The folk image has no answer for TDZ — it's not compatible with "code was rewritten." The creation-phase model explains it directly: stage 1 fires in creation (name exists), stage 2 waits for the declarator line (no value yet) — so reading the name in between finds the binding but trips the uninitialized check.

### 2. `let x = 5` is not `let x;` + `x = 5`

The natural "move the declaration up" rewrite would split `let x = 5;` into `let x;` at the top of scope and `x = 5;` at the original line. That's not equivalent:

```js
// Original
console.log(x); // ReferenceError — x in TDZ
let x = 5;

// Folk-image rewrite
let x; // stage 2 fires here → x is undefined
console.log(x); // "undefined" — not an error
x = 5;
```

`let x;` (with no initializer) still fires stage 2, with value `undefined`. So the rewrite **collapses the TDZ window** to zero, changing observable behavior. Equivalence holds only if nothing reads `x` between the top of the block and the original declarator line — a narrow special case, not the general rule.

On top of that, `let` is block-scoped, so "top of the scope" is ambiguous — the rewrite could silently promote a block-scoped binding to function scope, changing lifetime and shadowing.

### 3. Function declarations vs `var f = function() {}`

Looks similar; hoists very differently.

```js
hoistedFn(); // works — binding holds the function object
function hoistedFn() {
  return 1;
}

notHoistedFn(); // TypeError: notHoistedFn is not a function
var notHoistedFn = function () {
  return 1;
};
```

- `function foo() {}` is **one syntactic unit** — a function declaration. Creation phase registers the name _and_ initializes it to the function object in a single operation.
- `var foo = function() {}` is **two pieces** — a `var` declaration and an assignment statement. Creation handles the declarator part (stage 1+2 → `undefined`); the `= function() {...}` is an expression statement that only runs in the execution phase.

The RHS of a `var = ...` form could be any expression (`var x = computeIt()`). Since no code runs during creation, the engine can't evaluate it early — so all `var` declarators uniformly start as `undefined`, and any RHS waits for execution.

The error type is diagnostic: `ReferenceError` means "name doesn't exist or is in TDZ"; `TypeError` means "name exists, value doesn't support this operation." `notHoistedFn()` is a `TypeError` because the name is there — it just holds `undefined`.

### Why `class` is TDZ, not initialized

Classes behave like `let` for hoisting — declared-only during creation, TDZ until the declarator runs. Not like `function`, even though `class Dog {}` and `function Dog() {}` look textually parallel.

The reason comes back to "no user code runs during creation." A class body can contain:

- `extends` clause that's a function call: `class Sub extends computeParent() {}`
- Computed method names: `class C { [someName()]() {} }`
- Static field initializers: `class C { static x = expensive() }`

Evaluating any of those during creation would run user code, violating the axiom. Function declarations sidestep this because their only creation-phase operation is allocating a function object and packing the body code into its internal fields — the body itself doesn't run. A class body's surface-level expressions _must_ run to construct the class, so classes wait for execution.

## Precedence and duplicates

Two rules that come up in real code.

### Function declarations beat `var` during creation

If a `var` and a function declaration share a name in the same scope, the function wins during creation:

```js
console.log(typeof f); // "function" — not "undefined"
var f = 1;
function f() {}
```

After creation, `f` is the function object. The `var f` declarator is idempotent at stage 1 (name already registered) and doesn't overwrite with `undefined`. Then execution hits `var f = 1`, which runs as an assignment — `f` becomes `1`.

Source ordering of `var f = 1;` and `function f() {}` inside the scope doesn't matter — creation processes all declarations before any statement runs.

### Duplicate declarations

All parse-time errors where noted. A single `SyntaxError` prevents the _entire scope's code_ from running — not just the duplicate line.

| Combination                          | Result                          |
| ------------------------------------ | ------------------------------- |
| `var x` + `var x`                    | legal — single binding          |
| `var x` + `let x` (same scope)       | **`SyntaxError`** at parse time |
| `let x` + `let x` (same scope)       | **`SyntaxError`** at parse time |
| `let x` + `const x` (same scope)     | **`SyntaxError`** at parse time |
| `function f` + `function f` (strict) | last wins during creation       |

## `typeof` is not safe on TDZ names

`typeof` has historically been the one operator that doesn't throw `ReferenceError` on missing identifiers — the feature-detection escape hatch:

```js
if (typeof jQuery !== "undefined") {
  /* ... */
}
```

The escape hatch only applies when the name isn't declared _anywhere_ in the scope chain. Once a name exists — even if uninitialized — the escape hatch is gone:

```js
typeof neverDeclared; // "undefined" — safe, escape hatch applies
{
  typeof b; // ReferenceError — b exists in block ER, uninitialized
  let b;
}
```

Three distinct states a name can be in:

| State                 | When                                        | `typeof` on it   | Read on it       |
| --------------------- | ------------------------------------------- | ---------------- | ---------------- |
| **Undeclared**        | name never declared in any reachable scope  | `"undefined"`    | `ReferenceError` |
| **Declared, TDZ**     | stage 1 fired, stage 2 hasn't               | `ReferenceError` | `ReferenceError` |
| **Declared + init'd** | stage 2 done                                | actual type      | the value        |

The surprising row is the middle one — for declared-but-uninitialized bindings, `typeof` behaves like any other read. It's a rare case where declaring a name via `let` makes it _less_ safe to probe than leaving it undeclared.

## The formal names (spec-level)

If you want the spec terminology, the creation phase's setup operation has three names depending on the EC type:

- **Global Declaration Instantiation** — populates the Global ER on script/module start.
- **Function Declaration Instantiation** — populates a fresh Function ER on each function call.
- **Block Declaration Instantiation** — populates a block's Declarative ER on block entry.

All three walk the relevant AST region (the Abstract Syntax Tree — the engine's parsed representation of your source, produced once before any code runs), apply the per-keyword rule, and set the ER's bindings. They're the formal name for the "creation phase setup pass" from [creation-execution.md](creation-execution.md).

## Compressed takeaway

- **Hoisting = observable effect of the creation phase.** No code movement; the engine knows all names before line one runs because parsing already happened.
- **All declarations hoist the name.** What differs is the initial value during the window between scope entry and the declarator line.
- **"Hoisted-to-`undefined`" vs "in TDZ"** are two different states, both produced by the creation phase from different per-keyword rules.
- **`function foo() {}` ≠ `var foo = function() {}`.** First: name + value hoisted together. Second: only the name; value waits for execution. Error types diagnose the difference (`ReferenceError` vs `TypeError`).
- **Classes hoist like `let`, not like `function`** — class bodies can contain user code that can't run during creation.
- **Function declarations beat `var` of the same name in creation**; `let`/`const` duplicates (with anything of the same name) are parse-time `SyntaxError`s.
- **`typeof` loses its "safe" property for declared-but-uninitialized names** — TDZ reads throw on any read including `typeof`.
