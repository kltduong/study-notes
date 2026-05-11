# Scope Model — Draft

## The one question

Every scope rule in JS reduces to: **which ER instance does a binding land in?**

Two axes decide it:

1. **Declaration keyword** → picks the EC pointer used during creation.
   - `var` / `function` → `VariableEnvironment`
   - `let` / `const` / `class` → `LexicalEnvironment`

2. **Where that pointer is aimed** → depends on the scope boundary active when the declaration is processed.
   - `VariableEnvironment` is pinned at the function-or-global ER. Never moves.
   - `LexicalEnvironment` tracks the innermost scope — moves on every block entry/exit.

The combination produces four scope types. Not four arbitrary rules — four consequences of pointer routing.

---

## The four scope types

| Scope type      | Boundary that creates the ER                  | ER subtype                          | Keywords that land here                |
| --------------- | --------------------------------------------- | ----------------------------------- | -------------------------------------- |
| **Global**      | Script start                                  | Global ER (composite)               | `var`/`function` → ObjectRecord; `let`/`const`/`class` → DeclarativeRecord |
| **Function**    | Function call                                 | Function ER (extends Declarative)   | All local declarations (`var`, `let`, `const`, params) |
| **Block**       | `{ }`, `if`, `for`, `while`, `switch`, etc.   | Declarative ER                      | `let` / `const` / `class` / block-scoped `function` (strict) |
| **Module**      | Module evaluation                             | Module ER (extends Declarative)     | All declarations — nothing touches `globalThis` |

### Global scope — the composite router

You already know this from [execution-context.md](execution-context.md). The Global ER is unique: it's a composite that routes declarations to two sub-ERs.

```
Global ER
├── [[ObjectRecord]]       → Object ER backed by globalThis
├── [[DeclarativeRecord]]  → Declarative ER (hidden table)
└── [[VarNames]]           → bookkeeping
```

**Routing rule:**
- `var x` / `function f(){}` → `[[ObjectRecord]]` → becomes a property on `globalThis`
- `let y` / `const z` / `class C` → `[[DeclarativeRecord]]` → hidden, no `globalThis` property

This is why:
```js
var x = 1;
let y = 2;

globalThis.x; // 1 — lives on the object
globalThis.y; // undefined — not a property
```

Both are "global scope" — both are reachable from anywhere in the script. But they live in different sub-components with different visibility to the object layer.

**Why this matters practically:**

1. **Accidental globals.** `var` at top level pollutes `globalThis`. Third-party scripts in the same page can collide. `let`/`const` don't have this problem.
2. **`delete` behavior.** `var`-created properties are `configurable: false` — undeletable. Direct assignments (`globalThis.x = 1`) are `configurable: true` — deletable. The `[[VarNames]]` list tracks which is which.
3. **Cross-frame access.** `globalThis.x` is accessible from other frames (same-origin). `let y` is not — it's in a hidden table with no object-property reflection.

### Function scope — the `var` container

Every function call creates a fresh Function ER. `VariableEnvironment` is pinned to it for the lifetime of that call.

**What lands here:**
- `var` declarations (anywhere in the function body — including inside blocks)
- Function parameters
- `arguments` object
- `function` declarations (in sloppy mode; strict mode + block → block-scoped)

**What doesn't:**
- `let` / `const` inside blocks — those go to the Block ER via `LexicalEnvironment`
- `let` / `const` at function top level — technically also in the Function ER, because `LexicalEnvironment` starts there before any block is entered

The key insight: `var` always "escapes" to the function boundary because `VariableEnvironment` never moves. It doesn't matter how many blocks deep the `var` sits — it routes through the pinned pointer.

```js
function f() {
  // VariableEnvironment → Function ER (pinned)
  // LexicalEnvironment  → Function ER (starts here)

  if (true) {
    // LexicalEnvironment → new Block ER
    var x = 1;   // → VariableEnvironment → Function ER (bypasses block)
    let y = 2;   // → LexicalEnvironment  → Block ER
  }

  // LexicalEnvironment → back to Function ER
  console.log(x); // 1 — x is in Function ER
  console.log(y); // ReferenceError — y was in Block ER, now gone
}
```

### Block scope — `let`/`const` containers

Any `{ }` that contains `let`, `const`, `class`, or a strict-mode `function` declaration triggers a new Declarative ER at block entry.

**What creates a block scope:**
- Bare blocks `{ }`
- `if` / `else` bodies
- `for` / `while` / `do-while` bodies
- `switch` body (one ER for the whole switch, not per-case)
- `try` / `catch` / `finally` blocks

**What does NOT create a block scope:**
- Object literals `{ key: value }` — not a block, it's an expression
- Function bodies — those create a Function ER, not a Block ER
- Empty blocks with no `let`/`const`/`class` — the spec technically creates the ER, but it's observably empty

**The `for` loop special case:**

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// prints: 0, 1, 2
```

Each iteration gets a **fresh Block ER** with its own `i` binding, copied from the previous iteration's value. This is spec-mandated behavior for `for (let ...)` — the loop body's ER is recreated per iteration. It's why closures inside `for-let` loops capture the "current" value.

Contrast with `var`:
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// prints: 3, 3, 3
```

One `i` in the Function ER, shared across all iterations. All closures capture the same binding — which holds `3` by the time they run.

This per-iteration ER is the spec-level mechanism behind the classic "closure in a loop" problem and its `let` solution.

### Module scope — the isolation layer

A module (`<script type="module">` or a `.mjs` file) gets a Module ER — a subtype of Declarative ER.

**Critical difference from script-level global:**
- In a script: `var x = 1` → `globalThis.x === 1`
- In a module: `var x = 1` → `globalThis.x === undefined`

In a module, `VariableEnvironment` points to the Module ER (Declarative), not the Global ER's ObjectRecord. So `var` in a module behaves like `let` in terms of `globalThis` visibility — it doesn't attach.

```
Module EC
├── VariableEnvironment → Module ER (Declarative subtype)
├── LexicalEnvironment  → Module ER (same, initially)
└── [[OuterEnv]] of Module ER → Global ER
```

The Global ER is still reachable via the `[[OuterEnv]]` chain — so `console`, `setTimeout`, etc. resolve normally. But no module-level declaration creates a `globalThis` property.

**Why this matters:** modules are the default isolation mechanism. Each module is its own scope island — `var` pollution can't leak between modules or onto the global object. This is one reason the ecosystem moved to modules.

Additional Module ER feature: **import bindings are live links**, not copies. When the exporting module updates a binding, the importing module sees the new value. This is implemented as a special binding type in the Module ER that delegates reads to the source module's ER.

---

## Scope rules as pointer consequences

Everything above reduces to this table:

| Declaration | EC pointer used | Where that pointer aims | Consequence |
|---|---|---|---|
| `var` | `VariableEnvironment` | Function ER (or Global ER's ObjectRecord) | Function-scoped; attaches to `globalThis` in scripts |
| `function` (sloppy, top-level) | `VariableEnvironment` | Same as `var` | Same as `var` |
| `let` / `const` / `class` | `LexicalEnvironment` | Innermost ER (Block, Function, Module, or Global's DeclarativeRecord) | Block-scoped; never on `globalThis` |
| `function` (strict, in block) | `LexicalEnvironment` | Block ER | Block-scoped (like `let`) |

**The invariant that makes `var` function-scoped:** `VariableEnvironment` never moves. It's pinned at function (or global) entry and stays there. Any `var`, no matter how deeply nested in blocks, routes through this pinned pointer → lands in the function-level ER → visible throughout the function.

**The invariant that makes `let`/`const` block-scoped:** `LexicalEnvironment` moves on every block entry. A `let` inside a block routes through the moved pointer → lands in the Block ER → unreachable once the block closes and the pointer reverts.

---

## Global object attachment — why it's problematic

The `var` → `globalThis` attachment is a legacy design with real costs:

### 1. Namespace collision

```js
// library-a.js (loaded via <script>)
var config = { debug: true };

// library-b.js (loaded via <script>)
var config = { theme: "dark" };

// library-a's config is silently overwritten
```

Both `var config` declarations route to `[[ObjectRecord]]` → same property on `globalThis`. Last one wins. No error, no warning. `let` in a module avoids this entirely — each module has its own ER.

### 2. Implicit globals (sloppy mode)

In sloppy mode, assigning to an undeclared name creates a property on `globalThis`:

```js
function f() {
  oops = 42; // no var/let/const — creates globalThis.oops
}
f();
console.log(globalThis.oops); // 42
```

This isn't a `var` declaration — it's a failed name resolution that falls through to the Global ER's `[[ObjectRecord]]` and creates a property there (with `configurable: true`). Strict mode turns this into a `ReferenceError`.

### 3. `globalThis` is enumerable

```js
var x = 1;
for (let key in globalThis) {
  // x shows up here, alongside hundreds of browser APIs
}
```

`var`-created properties are `enumerable: true`. They mix with browser-provided properties in iteration. `let`/`const` bindings in the DeclarativeRecord are invisible to property enumeration — they're not properties at all.

### 4. The `with` statement (legacy, but illustrative)

`with (obj)` creates an Object ER backed by `obj`. Name resolution checks `obj`'s properties first. Combined with `var`-on-globalThis, this created security and optimization nightmares — the engine couldn't statically determine which ER a name would resolve in. Strict mode bans `with` entirely.

---

## Script vs module — the scope topology

Two different ER topologies depending on how code is loaded:

**Script (`<script>` or `<script defer>`):**
```
Global EC
├── VariableEnvironment → Global ER
└── LexicalEnvironment  → Global ER

Global ER (composite)
├── [[ObjectRecord]] → Object ER (globalThis)
├── [[DeclarativeRecord]] → Declarative ER
└── [[OuterEnv]] → null
```

**Module (`<script type="module">`):**
```
Module EC
├── VariableEnvironment → Module ER
└── LexicalEnvironment  → Module ER

Module ER (Declarative subtype)
└── [[OuterEnv]] → Global ER → null
```

The module adds one extra ER layer between your code and the Global ER. That layer is Declarative (hidden table), so nothing leaks to `globalThis`. The Global ER is still in the chain — you can still read `globalThis.setTimeout` — but you can't write to it via declarations.

---

## Compressed takeaway

- **Two axes decide scope:** declaration keyword (picks the pointer) × scope boundary (where the pointer aims).
- **Four scope types:** global, function, block, module — each is an ER instance created at a specific boundary.
- **`var` is function-scoped** because `VariableEnvironment` is pinned at the function ER and never moves.
- **`let`/`const` are block-scoped** because `LexicalEnvironment` moves to each new Block ER.
- **Global ER is a composite** — `var` routes to the Object ER (visible on `globalThis`), `let`/`const` route to the Declarative ER (hidden).
- **Module ER interposes** between your code and the Global ER — `var` in a module doesn't touch `globalThis`.
- **`for (let ...)` creates a fresh ER per iteration** — the spec-level mechanism behind the closure-in-loop fix.
- **Global object attachment is problematic:** namespace collisions, implicit globals, enumeration noise. Modules + `let`/`const` eliminate all three.
