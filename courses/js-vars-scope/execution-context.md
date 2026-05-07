# Execution Context Internals

**TL;DR:** Every piece of JS code runs inside an Execution Context (EC) — a spec-level structure holding everything needed to execute: which realm it belongs to, and pointers to Environment Records (ERs) where bindings live. Name resolution is a walk up the `[[OuterEnv]]` chain of ERs. The differences between `var`, `let`/`const`, and global declarations all reduce to _which ER_ a binding lands in and _how that ER stores it_.

## Two axioms everything follows from

1. **Every unit of code runs inside an Execution Context** — the EC is the complete environment needed to execute: what names are visible, what `this` is, what realm it belongs to. Concretely: an EC is what the call stack stores — each frame is an EC. See [Connection to async-js](#connection-to-async-js).
2. **Name resolution is a lookup in a chain of Environment Records** — each EC holds pointers to ERs, and each ER has an `[[OuterEnv]]` link forming a chain. Resolving a name = walking that chain.

Hoisting, TDZ, scope chains, closures, and `this`-binding all follow from these two. This note builds the container; [creation-execution.md](creation-execution.md) fills it with the step-by-step walkthrough.

> **Aside — what is a spec-level data structure?**
>
> An EC is a **spec-level data structure** — a formally defined shape with named fields that exists only in the ECMAScript specification's model of execution. It's not a JS object you can `console.log`, and it doesn't prescribe how engines implement it internally. V8 may compile the equivalent of an EC lookup into a single machine instruction with no struct ever allocated.
>
> The double-bracket notation (`[[Realm]]`, `[[OuterEnv]]`, `[[ThisValue]]`) marks internal fields that no JS code can directly access. They exist so the spec can define behavior unambiguously. Some have JS-visible reflections (e.g. `Object.getPrototypeOf()` exposes `[[Prototype]]`), but the slot itself is spec-level.

## Execution Context types

The spec defines three kinds:

| EC type      | Created when…                                | Example                              |
| ------------ | -------------------------------------------- | ------------------------------------ |
| **Global**   | Script/module starts executing               | Top-level code in a `<script>` tag   |
| **Function** | A function is called                         | `foo()`, `obj.method()`, `new Bar()` |
| **Eval**     | `eval()` is called (rare, mostly irrelevant) | `eval("var x = 1")`                  |

Every running program has exactly one _running_ EC at any moment (single-threaded engine). The stack of suspended ECs is the call stack from async-js — now you can see what each "frame" actually is.

## Components of an Execution Context

```
EC.Realm                = <Realm Record>              -- held directly
EC.LexicalEnvironment   = <pointer to an ER>          -- let/const/class/function declarations
EC.VariableEnvironment  = <pointer to an ER>          -- var declarations
EC.PrivateEnvironment   = <pointer to an ER>          -- (ES2022+) class #private fields
(Function ECs also have: Generator, ScriptOrModule, etc.)
```

`Realm` is a direct value — the EC holds the Realm Record itself. `LexicalEnvironment` and `VariableEnvironment` are **pointers** — they don't hold bindings directly, they point to Environment Record instances that do. The ERs are separate structures; the EC just holds references to them.

This pointer-vs-instance distinction matters: when you enter a block, `LexicalEnvironment` is updated to point to a new block ER — the pointer moves, the old ER doesn't change. `VariableEnvironment` never moves within a function. See [LexicalEnvironment vs VariableEnvironment](#lexicalenvironment-vs-variableenvironment--why-two-pointers) below.

## Realm Record — the universe a script lives in

A **Realm Record** is the complete JS universe for a script:

| Field              | What it is                                                                         |
| ------------------ | ---------------------------------------------------------------------------------- |
| `[[Intrinsics]]`   | The built-in objects for this realm (`Object.prototype`, `Array`, `Promise`, etc.) |
| `[[GlobalObject]]` | The global object — the same object as `globalThis`/`window`/`global`              |
| `[[GlobalEnv]]`    | The Global Environment Record (top-level scope)                                    |

Each `<iframe>` gets its own Realm. That's why `[] instanceof Array` can be `false` across frames — the `Array` in one realm is a different object than in another.

**How `globalThis` enters the picture:** `globalThis`, `window` (browser), and `global` (Node) are all the same object — `[[GlobalObject]]` from the Realm Record. The Global Environment Record uses this object as backing storage for certain declarations (`var`, `function`). How that routing works is covered in [Global Environment Record](#global-environment-record) below.

## Environment Records — where bindings actually live

An **Environment Record** (ER) is a spec-level structure that holds name→value bindings. The scope-to-ER mapping is strictly **1:1** — one scope creates exactly one ER, and that ER belongs to exactly one scope. No ER is ever shared across scopes or reused. The Global ER is no exception to this rule — it's still one ER for one scope. Its only wrinkle is _internal_: it's a composite that bundles two sub-ERs as storage components inside a single scope (details in [Global Environment Record](#global-environment-record)).

The spec defines a type hierarchy — think of it like a class inheritance hierarchy, where each subtype adds fields or changes storage behavior:

```mermaid
graph TD
    ER["Environment Record<br/>(abstract base)"]
    DER["Declarative ER<br/>(let, const, class, function, params)"]
    OER["Object ER<br/>(backed by an actual JS object)"]
    GER["Global ER<br/>(composite: declarative + object)"]
    FER["Function ER<br/>(adds this, arguments, new.target)"]
    MER["Module ER<br/>(adds import bindings)"]

    ER --> DER
    ER --> OER
    ER --> GER
    DER --> FER
    DER --> MER

    style ER fill:#555,stroke:#fff,color:#fff
    style DER fill:#46c,stroke:#fff,color:#fff
    style OER fill:#c64,stroke:#fff,color:#fff
    style GER fill:#5a5,stroke:#fff,color:#fff
    style FER fill:#46c,stroke:#fff,color:#fff
    style MER fill:#46c,stroke:#fff,color:#fff
```

### Declarative Environment Record

The normal case. Holds bindings in an internal hidden table — not a JS object, not enumerable, not accessible via any API. `let`, `const`, function params, `class` declarations all go here. Function scopes and block scopes both use this type.

### Object Environment Record

A thin spec-level wrapper around a real JS object. It holds a `[[BindingObject]]` field — a reference to the JS object it delegates to. Every ER operation forwards to a property operation on that object:

| ER operation        | Property operation on `[[BindingObject]]` |
| ------------------- | ----------------------------------------- |
| Create binding `x`  | `DefineOwnProperty("x", ...)`             |
| Set `x = 1`         | `Set("x", 1)`                             |
| Read `x`            | `Get("x")`                                |
| Check if `x` exists | `HasProperty("x")`                        |

No hidden table — the binding table _is_ the object's properties. Used for `with` statements (legacy, ignore) and the object component of the Global ER, where `[[BindingObject]]` is `globalThis`.

### Global Environment Record

A composite — not itself backed by a JS object, but a router that delegates to two sub-ER instances:

```
Global Environment Record  (composite router, not itself an object)
├── [[DeclarativeRecord]]  →  instance of Declarative ER (hidden table)
├── [[ObjectRecord]]       →  instance of Object ER backed by globalThis
└── [[VarNames]]           →  bookkeeping list (see below)
```

The Global ER never stores bindings directly. Every operation (create, read, write) is forwarded to one sub-ER or the other based on the declaration keyword.

`[[DeclarativeRecord]]` and `[[ObjectRecord]]` are **fields on the Global ER**, each holding an instance of the corresponding type (Declarative ER and Object ER respectively). Bindings live inside those instances:

- `let x` at global scope → lives in the **Declarative ER instance** referenced by `[[DeclarativeRecord]]`.
- `var y` at global scope → lives in the **Object ER instance** referenced by `[[ObjectRecord]]` — backed by `globalThis`.

At global scope, declarations are routed to one component or the other based on keyword:

| Declaration         | Routed to               | Consequence                              |
| ------------------- | ----------------------- | ---------------------------------------- |
| `var x = 1`         | `[[ObjectRecord]]`      | Property on `window` → `window.x === 1`  |
| `function foo() {}` | `[[ObjectRecord]]`      | `window.foo` exists                      |
| `let y = 2`         | `[[DeclarativeRecord]]` | Hidden table — `window.y` is `undefined` |
| `const z = 3`       | `[[DeclarativeRecord]]` | Hidden table — `window.z` is `undefined` |
| `class C {}`        | `[[DeclarativeRecord]]` | Hidden table — `window.C` is `undefined` |

This is why `var x = 1` at global level creates `window.x` but `let y = 2` does not — they land in different components of the same Global ER.

> **Aside — why the composite design?** In ES3, all global declarations were `var`/`function` and the global scope _was_ the global object. When ES6 added `let`/`const`, they needed block scoping and TDZ — which don't work if the binding is a plain object property. So the Declarative ER component was added alongside the existing Object ER. Legacy declarations still route to the object (backward compat); new ones route to the hidden table.

### Function Environment Record

Extends Declarative ER with `[[ThisValue]]`, `arguments`, and `new.target`. Created fresh on every function call — each call gets its own instance. When the function returns, the EC is popped and the ER becomes unreachable, eligible for GC. Exception: if a closure captures a variable, it holds a reference to the ER, keeping it alive. See [scope-lexical.md](scope-lexical.md).

Inside a function, no Object ER exists in the local scope — declarations here never create properties on `globalThis`. All local bindings land in Declarative ERs. The difference is which one:

- `var` → Function ER (via `VariableEnvironment`, never moves)
- `let`/`const` → current block ER (via `LexicalEnvironment` — which is the Function ER itself if there's no enclosing block)

### Module Environment Record

Extends Declarative ER with import bindings — live links to the exporting module's bindings (not copies).

## The `[[OuterEnv]]` chain

Every ER has an `[[OuterEnv]]` field — either `null` (Global ER, the top) or a reference to the enclosing ER. This is the scope chain.

```mermaid
graph LR
    Block["Block ER<br/>(let z)"] -->|"[[OuterEnv]]"| Func["Function ER<br/>(let y, var x)"]
    Func -->|"[[OuterEnv]]"| Global["Global ER<br/>(var g)"]
    Global -->|"[[OuterEnv]]"| Null["null"]

    style Block fill:#46c,stroke:#fff,color:#fff
    style Func fill:#46c,stroke:#fff,color:#fff
    style Global fill:#5a5,stroke:#fff,color:#fff
    style Null fill:#555,stroke:#fff,color:#fff
```

Name resolution: start at the current ER, check if the name exists. If not, follow `[[OuterEnv]]` and repeat. Hit `null` → `ReferenceError`. This is the formal mechanism behind lexical scoping — the chain is fixed at the point where a function is _defined_, not where it's _called_. Full treatment in [scope-lexical.md](scope-lexical.md).

## LexicalEnvironment vs VariableEnvironment — why two pointers?

In a simple function with no blocks, both pointers point to the same Function ER. They diverge when a block is entered:

```js
function outer() {
  // L1
  var a = 1; // L2
  let b = 2; // L3
  {
    // L4 — block starts
    var c = 3; // L5
    let d = 4; // L6
  } // L7 — block ends
}
```

When `outer()` is called:

- **VariableEnvironment** → Function ER (will hold `a` and `c`)
- **LexicalEnvironment** → same Function ER (holds `b`)

When execution enters the block at L4:

- A new Declarative ER is created for the block, `[[OuterEnv]]` → Function ER
- **LexicalEnvironment** updates to point to the new block ER (holds `d`)
- **VariableEnvironment** stays at the Function ER

`var c` at L5 goes into the Function ER via VariableEnvironment. `let d` at L6 goes into the block ER via LexicalEnvironment. When the block ends at L7, LexicalEnvironment reverts to the Function ER.

**The invariant:** VariableEnvironment never moves within a function. LexicalEnvironment tracks current block depth. This is the mechanism behind `var` being function-scoped and `let`/`const` being block-scoped — they're stored via different pointers.

## `[[VarNames]]` and the `delete` behavior

The Global ER maintains a `[[VarNames]]` list — all names created via `var` or `function` at global level.

Every property on a JS object has a **property descriptor** with a `configurable` flag:

- `configurable: true` → `delete` works, property can be redefined
- `configurable: false` → `delete` returns `false`, property is permanent

When `var x = 1` routes to the Object ER (backed by `globalThis`), the spec creates the property with `configurable: false` — a declaration signals intent for a stable binding. A plain assignment (`window.y = 1`) uses `configurable: true`.

```js
var x = 1;
window.y = 1;

Object.getOwnPropertyDescriptor(window, "x");
// { value: 1, writable: true, enumerable: true, configurable: false }

Object.getOwnPropertyDescriptor(window, "y");
// { value: 1, writable: true, enumerable: true, configurable: true }

delete window.x; // false — non-configurable
delete window.y; // true
```

`[[VarNames]]` is the spec's own internal record on top of this — used by re-declaration checks and `eval` conflict detection algorithms that operate at the spec level, not the property level.

## Connection to async-js

From async-js: each function call pushes a frame onto the call stack, return pops it. That "frame" is an Execution Context — carrying its Realm pointer and its ER pointers. When `await` suspends a function, this entire EC structure is shelved and later restored. The call stack is a stack of ECs.
