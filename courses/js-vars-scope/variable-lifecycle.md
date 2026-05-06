# Variable Lifecycle — The 3-Stage Model

**TL;DR:** Every JS variable goes through three stages: declaration (name registered), initialization (first value), assignment (subsequent values). The differences between `var`, `let`, and `const` reduce to _when_ each stage fires and _whether_ stage 3 is allowed. The gap between stages 1 and 2 for `let`/`const` is the TDZ (Temporal Dead Zone).

## Two phases (just enough to read the table)

When a scope is entered, the engine works in two passes:

1. **Creation phase** — scans the scope for declarations and sets up bindings. No code runs yet.
2. **Execution phase** — runs statements top-to-bottom. Assignments, function calls, expressions happen here.

The full machinery behind these phases (execution contexts, Environment Records) is covered in [execution-context.md](execution-context.md) and [creation-execution.md](creation-execution.md). For now, "creation = setup pass, execution = runtime pass" is enough.

## The three stages

Every variable goes through up to three stages in order: declaration → initialization → assignment. Not every keyword permits all three, and they fire at different times — that's the entire source of `var`/`let`/`const` behavioral differences.

| Stage              | What happens                                                                                                                                                                               |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Declaration**    | The engine registers the name in the current scope's Environment Record. The binding exists but has no value — not even `undefined`.                                                       |
| **Initialization** | The binding receives its first value. For `var` this is auto-initialized to `undefined`; for `let`/`const` it stays uninitialized until the engine reaches the declarator in source order. |
| **Assignment**     | A new value is written to an already-initialized binding.                                                                                                                                  |

## Why separate declaration from initialization?

This is the design choice that makes TDZ possible. Follow the chain:

### 1. JS has features that force pre-registration

JS resolves scope _statically_ — every name-to-scope binding is determined before any code runs. This isn't optional; two features force it:

- **Hoisting** (`var` + function declarations) — names (and for functions, values) must be available before their textual position. The engine must know a `var` name belongs to the function even if its declarator sits inside an `if (false)` branch. See [hoisting.md](hoisting.md).
- **Block-scoped shadowing** — `{ console.log(x); let x = 2; }` must know at compile time that the inner `x` shadows any outer `x`. See [scope-lexical.md](scope-lexical.md).

Both need the engine to "know what names exist where" before execution starts. That's what the **creation phase** does — it walks the scope and registers every declared name.

### 2. The creation phase leaves a window

Once names are pre-registered, there's a window between "name exists" (end of creation) and "value assigned" (execution reaches the declarator line). Something has to happen if code reads the binding during that window. Two options:

- **Give it a default value** — the `var` approach: auto-initialize to `undefined`. No window, no errors, but silent bugs slip through:

  ```js
  if (!user) {
    // L1 — runs, because user is undefined
    redirectToLogin(); // L2 — wrong branch taken
  }
  var user = getUser(); // L3
  ```

  Python doesn't have this problem because it throws `NameError` on use-before-assignment — but Python has no creation phase to leave a window in.

- **Throw on access** — the `let`/`const` approach: the binding exists but is marked uninitialized. Any read in the window throws `ReferenceError: Cannot access 'x' before initialization`.

### 3. TDZ is option 2

TDZ (Temporal Dead Zone) is the name for that "declared but uninitialized" window. It's how JS gets Python-like use-before-init errors while keeping the creation phase that hoisting and block shadowing require. Full treatment in [tdz.md](tdz.md).

The lifecycle mapping falls out of this:

- `var` → option 1. Stages 1+2 collapse during creation; the window is filled with `undefined`.
- `let`/`const` → option 2. Only stage 1 runs in creation; stage 2 is deferred; the window is the TDZ.

**The principle:** JS is a _statically scoped language with dynamic execution_. Static scoping requires the creation phase. The creation phase creates a window. TDZ is the cheapest mechanism for making that window safe.

## How each keyword maps to the stages

| Keyword | Stage 1 (declaration) | Stage 2 (initialization)             | Stage 3 (assignment)        |
| ------- | --------------------- | ------------------------------------ | --------------------------- |
| `var`   | Creation phase        | Creation phase (auto → `undefined`)  | Execution phase, repeatable |
| `let`   | Creation phase        | Execution phase (at declarator line) | Execution phase, repeatable |
| `const` | Creation phase        | Execution phase (at declarator line) | **Never** — `TypeError`     |

`const` and `let` are identical through stages 1 and 2. The only difference: `const` makes stage 3 permanently forbidden.

## Worked example — all three keywords

```js
console.log(a); // undefined                          // L1
console.log(b); // ReferenceError (TDZ)               // L2
console.log(c); // ReferenceError (TDZ)               // L3

var a = 1; // L4 — assignment (stage 3)
let b = 2; // L5 — initialization (stage 2). TDZ ends.
const c = 3; // L6 — initialization (stage 2). No stage 3 ever.
```

**Creation phase** scans the scope:

| Binding | Stage 1       | Stage 2                    | State after creation        |
| ------- | ------------- | -------------------------- | --------------------------- |
| `a`     | ✅ registered | ✅ auto-set to `undefined` | usable (value: `undefined`) |
| `b`     | ✅ registered | ❌ deferred                | in TDZ                      |
| `c`     | ✅ registered | ❌ deferred                | in TDZ                      |

**Execution phase** runs top-to-bottom:

- L1: `a` is initialized → reads `undefined`.
- L2: `b` is uninitialized → `ReferenceError`.
- L3: `c` is uninitialized → `ReferenceError` (same as `let` in TDZ).
- L4: assigns `1` to `a` (stage 3).
- L5: initializes `b` to `2` (stage 2).
- L6: initializes `c` to `3` (stage 2). Any later `c = 5` → `TypeError`.

## `let` moving through all three stages

```js
// --- Creation phase has already run ---
// `x` is declared (stage 1 done), but uninitialized. TDZ active.

console.log(x); // L1 — ReferenceError: x is in TDZ (stage 1 only)

let x = 10; // L2 — initialization to 10 (stage 2). TDZ ends.

console.log(x); // L3 — 10

x = 20; // L4 — assignment (stage 3). New value written.

console.log(x); // L5 — 20
```

| Point                | Stage           | Binding state         | Access result    |
| -------------------- | --------------- | --------------------- | ---------------- |
| After creation phase | 1 (declared)    | exists, uninitialized | `ReferenceError` |
| After L2 executes    | 2 (initialized) | holds `10`            | `10`             |
| After L4 executes    | 3 (assigned)    | holds `20`            | `20`             |

Stage 3 is repeatable — each `x = ...` writes a new value to the already-initialized binding.

## The mental model

```mermaid
graph LR
    A["Declaration<br/>(name registered)"] --> B["Initialization<br/>(first value)"]
    B --> C["Assignment<br/>(subsequent values)"]

    style A fill:#c9a,stroke:#fff,color:#fff
    style B fill:#46c,stroke:#fff,color:#fff
    style C fill:#3a7,stroke:#fff,color:#fff
```

- `var`: A and B happen together at creation. C at runtime.
- `let`: A at creation. B at the declarator line. C anytime after.
- `const`: A at creation. B at the declarator line. C **never**.

The gap between A and B for `let`/`const` _is_ the TDZ. No gap for `var` → no TDZ.

Everything else in this course — hoisting, TDZ details, scope rules — will be consequences of _where_ and _when_ these three stages fire for each declaration kind.
