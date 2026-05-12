# Scope Model

**TL;DR:** Every scope rule reduces to one question — which Environment Record (ER) does a binding land in? Two axes decide it: the declaration keyword (picks the EC pointer) × where that pointer is aimed (depends on the active scope boundary).

## The one question

Every scope rule in JS reduces to: **which ER instance does a binding land in?**

Two axes decide it:

1. **Declaration keyword** → picks the EC pointer used during creation.
   - `var` / `function` → `VariableEnvironment`
   - `let` / `const` / `class` → `LexicalEnvironment`

2. **Where that pointer is aimed** → depends on the scope boundary active when the declaration is processed.
   - `VariableEnvironment` is pinned at the function-or-global ER. Never moves.
   - `LexicalEnvironment` tracks the innermost scope — moves on every block entry/exit.

> **Aside —** "Aimed at" = which ER instance the pointer currently references. `LexicalEnvironment` changes its target on every block entry/exit. `VariableEnvironment` never changes its target after the function (or script) EC is created.

The combination produces four scope types. Not four arbitrary rules — four consequences of pointer routing.

---

## The four scope types

| Scope type | Boundary | Pointer used | ER subtype | Keywords |
| --- | --- | --- | --- | --- |
| **Global** | Script start | Both (routes internally) | Composite: `[[ObjectRecord]]` + `[[DeclarativeRecord]]` | `var`/`function` → ObjectRecord; `let`/`const`/`class` → DeclarativeRecord |
| **Function** | Function call | `VariableEnvironment` (pinned here) | Function ER (extends Declarative) | All locals: `var`, `let`, `const`, params |
| **Block** | `{ }`, `if`, `for`, `while`, `switch`, … | `LexicalEnvironment` (moved here) | Declarative ER | `let` / `const` / `class` / strict block `function` |
| **Module** | Module evaluation | Both (but `VariableEnvironment` aims at Module ER, not Global) | Module ER (extends Declarative) | All declarations — nothing reaches `globalThis` |

### Global scope — the composite router

See [execution-context.md](execution-context.md) for the full Global ER structure. The key point here: the Global ER is unique in that both pointers aim at the same ER, but the ER itself routes declarations internally to two sub-records based on keyword.

```
Global ER
├── [[ObjectRecord]]       → Object ER backed by globalThis
├── [[DeclarativeRecord]]  → Declarative ER (hidden table)
└── [[VarNames]]           → bookkeeping
```

**Routing rule:**
- `var x` / `function f(){}` → `[[ObjectRecord]]` → becomes a property on `globalThis`
- `let y` / `const z` / `class C` → `[[DeclarativeRecord]]` → hidden, no `globalThis` property

```js
var x = 1;
let y = 2;

globalThis.x; // 1 — lives on the object
globalThis.y; // undefined — not a property
```

Both are "global scope" — both reachable from anywhere in the script. But they live in different sub-components with different visibility to the object layer.

### Function scope — the `var` container

Every function call creates a fresh Function ER. `VariableEnvironment` is pinned to it for the lifetime of that call.

**What lands here:**
- `var` declarations (anywhere in the function body — including inside blocks)
- `let` / `const` / `class` at function top level (no enclosing block — `LexicalEnvironment` still aims at the Function ER here)
- Function parameters
- `arguments` object
- `function` declarations (in sloppy mode; strict mode + block → block-scoped)

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

Each iteration gets a **fresh Block ER** with its own `i` binding, copied from the previous iteration's value. This is spec-mandated behavior for `for (let ...)` — the loop body's ER is recreated per iteration.

Contrast with `var`:
```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// prints: 3, 3, 3
```

One `i` in the Function ER, shared across all iterations. But the shared binding alone doesn't force `3, 3, 3` — *when* each read happens does. `setTimeout` queues the arrow; it doesn't fire until the call stack is empty, which is after the loop has exited and `i++` has run its final increment (loop exits with `i === 3`, not `2` — the check `3 < 3` is what ends it). Each queued arrow then reads `i` via `[[OuterEnv]]` at call time and sees `3`. Swap the `setTimeout(...)` for a bare `console.log(i)` and the reads move inline into the loop body → `0, 1, 2`. Same binding, different timing, different output.

**Body-mutated variables — a subtler trap:**

```js
for (var i = 0; i < 3; i++) {
  var x = i * 10;
  setTimeout(() => console.log(x), 0);
}
// prints: 20, 20, 20  — not 30, 30, 30
```

`x` is also a single `var` binding in the Function ER, but its final value is `20`, not `30`. The difference: `x = i * 10` is a **body statement** — it only runs when the condition passes. When `i === 3`, the check `3 < 3` fails and the body is skipped entirely. So `x` freezes at the last successful iteration's write (`2 * 10 = 20`). The `for` loop's step order is `init → cond → body → update → cond → ...` — the update (`i++`) runs one more time than the body does.

General rule: after a `for` loop, a body-mutated variable holds whatever the **last executed iteration** set it to, not whatever the loop's "exit state" suggests. `i` advances past the body because `i++` is in the update step; `x` doesn't because nothing writes to it outside the body.

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

**Import bindings are live links**, not copies. When the exporting module updates a binding, the importing module sees the new value — implemented as a special binding type in the Module ER that delegates reads to the source module's ER.

---

## Worked example — tracing every binding

With all four scope types in hand, the snippet below puts them in one place. For each declaration, the trace answers: which pointer did the engine use, where was that pointer aimed, and which ER did the binding land in?

```js
// ─── Script context (not a module) ───────────────────────────────────────────
// Active EC:  Global EC
//   VariableEnvironment → Global ER  (pinned here for the script's lifetime)
//   LexicalEnvironment  → Global ER  (starts here; moves on block entry)

var globalVar = "gv";
// Keyword: var → VariableEnvironment → Global ER → [[ObjectRecord]]
// Result: globalThis.globalVar === "gv"

let globalLet = "gl";
// Keyword: let → LexicalEnvironment → Global ER → [[DeclarativeRecord]]
// Result: reachable globally, but NOT a property on globalThis

function outer(param) {
  // ─── Function call: new Function EC created ─────────────────────────────
  // Active EC:  outer's EC
  //   VariableEnvironment → outer's Function ER  (pinned for this call)
  //   LexicalEnvironment  → outer's Function ER  (starts here)

  var funcVar = "fv";
  // Keyword: var → VariableEnvironment → outer's Function ER
  // Result: visible anywhere inside outer(), including inside nested blocks

  let funcLet = "fl";
  // Keyword: let → LexicalEnvironment → outer's Function ER
  // (LexicalEnvironment still aims at the Function ER — no block entered yet)
  // Result: same lifetime as funcVar here, but for a different reason:
  //         funcLet lands in Function ER only because no block is active yet

  // param lands in outer's Function ER alongside funcVar (same ER, same pointer)

  if (true) {
    // ─── Block entry: new Block ER pushed ───────────────────────────────
    // LexicalEnvironment moves → new Block ER (child of outer's Function ER)
    // VariableEnvironment stays → outer's Function ER (never moves)

    var blockVar = "bv";
    // Keyword: var → VariableEnvironment → outer's Function ER  (bypasses block)
    // Result: same binding as if declared at the top of outer()

    let blockLet = "bl";
    // Keyword: let → LexicalEnvironment → Block ER
    // Result: destroyed when the block exits; ReferenceError outside

    const blockConst = "bc";
    // Keyword: const → LexicalEnvironment → Block ER  (same as let)
    // Result: same lifetime as blockLet; additionally immutable after init

    function inner() {
      // ─── Nested function call: new Function EC ────────────────────────
      // Active EC:  inner's EC
      //   VariableEnvironment → inner's Function ER  (fresh, pinned)
      //   LexicalEnvironment  → inner's Function ER  (starts here)
      // inner's Function ER [[OuterEnv]] → Block ER → outer's Function ER → Global ER

      var innerVar = "iv";
      // Keyword: var → VariableEnvironment → inner's Function ER
      // Result: local to inner(); invisible to outer()

      console.log(funcVar);   // resolves via [[OuterEnv]] chain: inner ER → Block ER → outer ER ✓
      console.log(blockLet);  // resolves via [[OuterEnv]] chain: inner ER → Block ER ✓
      console.log(globalVar); // resolves all the way up to Global ER's [[ObjectRecord]] ✓
    }

    inner();
    // ─── Block exit ──────────────────────────────────────────────────────
    // LexicalEnvironment reverts → outer's Function ER
    // Block ER is discarded; blockLet and blockConst are gone
  }

  console.log(funcVar);  // ✓ — in outer's Function ER
  console.log(blockVar); // ✓ — also in outer's Function ER (var bypassed the block)
  // console.log(blockLet); // ✗ ReferenceError — Block ER is gone
}

outer("p");
```

**Pointer state at each scope boundary:**

| Location in code | `VariableEnvironment` aims at | `LexicalEnvironment` aims at |
|---|---|---|
| Script top level | Global ER | Global ER |
| Inside `outer()`, before `if` | outer's Function ER | outer's Function ER |
| Inside `if` block | outer's Function ER | Block ER |
| Inside `inner()` | inner's Function ER | inner's Function ER |
| Back in `outer()`, after `if` | outer's Function ER | outer's Function ER (reverted) |

The `VariableEnvironment` column never changes within a given function call. The `LexicalEnvironment` column is the one that moves.

---

## Scope rules as pointer consequences

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

> **Aside — Node.js.** The script/module split above is browser-framing. In Node.js, CommonJS files (`.js` with `"type": "commonjs"`) are wrapped in a function by the module loader — `(function(exports, require, module, __filename, __dirname) { … })` — so your "top-level" code is actually inside a function call. `var x` lands in that wrapper's Function ER, not the global `[[ObjectRecord]]`, and `global.x` stays `undefined`. Node ES modules (`.mjs` or `"type": "module"`) use the Module ER topology above — same isolation as browser modules. The underlying spec concepts (Global ER, Module ER, ER chain) are engine-level and apply everywhere; the `<script>` entry point is browser-specific.

---

## Takeaway

- **Two axes decide scope:** declaration keyword (picks the pointer) × where that pointer is aimed (the active scope boundary).
- **`var` is function-scoped** because `VariableEnvironment` is pinned at the function ER and never moves.
- **`let`/`const` are block-scoped** because `LexicalEnvironment` moves to each new Block ER.
- **Global ER is a composite** — `var` routes to the Object ER (visible on `globalThis`), `let`/`const` route to the Declarative ER (hidden).
- **Module ER interposes** between your code and the Global ER — `var` in a module doesn't touch `globalThis`.
- **`for (let ...)` creates a fresh ER per iteration** — the spec-level mechanism behind the closure-in-loop fix.
- **Global object attachment is problematic:** namespace collisions, implicit globals, enumeration noise. Modules + `let`/`const` eliminate all three.
