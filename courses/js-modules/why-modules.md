# Why modules exist

**TL;DR.** Pre-module JavaScript runs every `<script>` into one shared global object — `window` in the browser. That single mechanism creates four independent pain points: name collisions, implicit load order, no encapsulation, no explicit dependency declaration. The IIFE namespace pattern (a function called immediately, returning a public-API object) closes problems 1 and 3 with pure language features, but can't touch 2 and 4 — those need a *loader*, which has to live in the **host** (browser, Node, bundler) because JavaScript itself has no fetching primitive. A real module system is therefore a partnership: language-level syntax for declaring scope, exports, and dependencies; host-level machinery for fetching and ordering. Two independent design axes shape any concrete system — *where fetching lives* (always the host) and *when the dependency graph is known* (runtime, à la CommonJS `require()`, vs static, à la ES modules' `import`). ESM picks static, and that single choice unlocks tree-shaking, parallel network fetch, top-level `await`, and parse-time error catching — all consequences of the dep graph being a decidable static property, not a runtime emergent.

> **Aside —** *Module* here means *a single file with its own scope, an explicit public API, and named dependencies on other files.* *Loader* means *the component that fetches files, orders them by dependency, and feeds them to the engine.* *Host* means *the runtime environment around the JS engine — browser, Node, bundler — providing capabilities the language doesn't (fetching, timers, DOM, filesystem).*

---

## The pre-module world

### One shared global

The pre-module mechanism is a single rule:

> Every `<script>` tag's top-level `var` and `function` declarations become **properties of the global object** (`window` in browsers, `global` / `globalThis` in Node's classic-script mode).

Multiple `<script>` tags don't run in separate scopes that get "merged" — they each contribute declarations to the same shared object from the start. The host fetches each file, hands it to the engine as a *Script Record*, and the engine parses + executes it; whatever sticks at top level lives on the global from then on.

```html
<script src="formatter.js"></script>  <!-- L1 -->
<script src="chart.js"></script>      <!-- L2 -->
<script src="app.js"></script>        <!-- L3 -->
```

```js
// formatter.js
var pi = 3.14;                                    // L1 — becomes window.pi
function area(r) { return pi * r * r; }           // L2 — becomes window.area
```

```js
// app.js
console.log(area(5));        // L1 — 78.5
console.log(window.pi);      // L2 — 3.14 (var landed on window)
console.log(window.area);    // L3 — [Function: area]
```

That `window.pi === 3.14` isn't a quirk — it's the *defining* mechanism. Four problems fall out of it directly.

### Problem 1 — Name collisions

Two files declaring the same top-level name silently overwrite each other on the global object.

```js
// formatter.js
function format(n) { return `$${n}`; }            // L1 — becomes window.format

// chart.js
function format(d) { return d.toFixed(2); }       // L1 — overwrites window.format silently
```

No error. No warning. The second `<script>` wins because it ran later. The collision surface is *every* top-level declaration in *every* file you load — including third-party libraries.

### Problem 2 — Implicit load order

The HTML `<script>` order is the de facto dependency declaration. `chart.js` works only because `formatter.js` ran first and parked `formatCurrency` on `window`. The dependency itself — `chart.js → formatter.js` — is invisible to the loader; the HTML author has to know it.

```html
<script src="chart.js"></script>             <!-- L1 -->
<script>renderChart(myData, ...);</script>   <!-- L2 — runs before formatter.js -->
<script src="formatter.js"></script>         <!-- L3 -->
```

L2 → calls `renderChart` → which calls `formatCurrency` → `TypeError: formatCurrency is not a function`.

> **Aside —** function declarations defer the lookup until call time, so `chart.js` can *parse* fine even when loaded first. Top-level statements that *use* free names break immediately. This is the same scope-chain lookup mechanism from the variables-and-scope course; nothing module-specific.

### Problem 3 — No encapsulation

Every top-level name is public. There's no language-level way to mark a helper as internal.

```js
// chart.js
function _pickAxisScale(data) { /* internal! */ }  // L1 — actually window._pickAxisScale, fully public
function renderChart(data, container) {            // L2
  const scale = _pickAxisScale(data);              // L3
}                                                  // L4
```

The leading underscore is a *convention*, not a barrier. And the failure mode isn't conditional on someone looking up `window._pickAxisScale` — they can write `_pickAxisScale` directly and hit the same global. Worse, *any* module can write `pi = 999` and silently break `formatter.js`'s `area` function — the encapsulation failure is dormant even when the program "works."

### Problem 4 — No explicit dependency declaration

In `chart.js`, `formatCurrency` is a **free reference** — syntactically indistinguishable from `parseInt`, `Math.PI`, or anything a third-party `<script>` parked on `window`. The language gives no way to express "this comes from `formatter.js`." A static analyzer reading `chart.js` alone has no idea where `formatCurrency` should resolve.

File count affects how *bad* problem 4 gets, not whether it's present. Two files makes it tractable for a human; 200 files makes it a forensics exercise.

### The four problems are independent

| Problem                            | Symptom                                       | What "fixing" it alone would require                          |
| ---------------------------------- | --------------------------------------------- | ------------------------------------------------------------- |
| Name collisions                    | Silent overwrite, last-script-wins            | Per-file scope (file is not part of global)                   |
| Implicit load order                | `TypeError`s when scripts ordered wrong       | A way for code to *declare* what it needs, plus a loader      |
| No encapsulation                   | Internal helpers leak as public globals       | Per-file scope + opt-in "this is the public API" mechanism    |
| No explicit dependency declaration | Free references everywhere, no static signal  | Syntax that names which file each external comes from         |

Per-file scope alone fixes 1 and partially fixes 3. Explicit `import` syntax fixes 4 and gives a loader the info to fix 2. So a real solution needs **at minimum two pieces together** — per-file scope *and* declarative import/export. Either alone is a half-fix.

---

## The IIFE workaround

Before ES2015, the community shipped a partial fix using only functions and lexical scoping. No new syntax — just a careful pattern.

### The pattern

Wrap the file in a function expression and *immediately call* it. The function body becomes a private scope; only what the function returns escapes.

```js
// formatter.js
var Formatter = (function () {                    // L1 — A: open IIFE, assign result to one global
  var locale = "en-US";                            // L2 — private to the IIFE
  function pad(n) { return String(n).padStart(2, "0"); }  // L3 — private helper

  function formatCurrency(n) {                     // L4
    return new Intl.NumberFormat(locale, { style: "currency", currency: "USD" }).format(n);  // L5
  }                                                // L6
  function formatDate(d) {                         // L7
    return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`;  // L8
  }                                                // L9

  return {                                         // L10 — B: explicit public API
    formatCurrency: formatCurrency,                // L11
    formatDate: formatDate,                        // L12
  };                                               // L13
})();                                              // L14 — A': close + invoke
```

`A` and `A'` together are the IIFE — *Immediately-Invoked Function Expression*. The parens around `function () { ... }` make it an expression (not a declaration), and the trailing `()` calls it on the spot. The return value at `B` is what gets assigned to `Formatter`.

```js
// chart.js
var Chart = (function () {                              // L1
  function pickAxisScale(data) { /* private */ }        // L2 — actually private now
  function renderChart(data, container) {               // L3
    const label = Formatter.formatCurrency(data[0]);    // L4 — explicit reach into formatter
  }                                                     // L5
  return { renderChart: renderChart };                  // L6
})();                                                   // L7
```

### What it fixes

**Problem 1 — collisions** ⚠️ Mitigated. One global per file (`Formatter`, `Chart`) instead of one per function. Nested namespacing (`MyCompany.UI.Chart`) pushed collision risk further down. But the mechanism is unchanged — there's still one shared global object underneath.

**Problem 3 — encapsulation** ✅ Fixed. `locale`, `pad`, `pickAxisScale` live in the IIFE's Function ER, never touching the global. The returned functions close over that ER (the `[[Environment]]` slot points at it), so it stays alive after the IIFE returns — same closure mechanism as counter-factory examples from the variables-and-scope course. The returned object's properties are the *intentional* public API.

### What it doesn't fix

**Problem 2 — implicit load order** ❌ Untouched. `chart.js` still references `Formatter` as a free name. Load order in HTML is still load-bearing.

**Problem 4 — no explicit dep declaration** ❌ Untouched. The dependency on `Formatter` is buried inside a function body. A static analyzer reading `chart.js` alone still can't tell where `Formatter` is supposed to come from.

### The dependency-injection variant

Closer to a real `import`: pass the dependency *into* the IIFE explicitly.

```js
// chart.js
var Chart = (function (Formatter) {                       // L1 — Formatter is a parameter
  function renderChart(data, container) {                 // L2
    const label = Formatter.formatCurrency(data[0]);      // L3 — uses the parameter, not a free name
  }                                                       // L4
  return { renderChart: renderChart };                    // L5
})(window.Formatter);                                     // L6 — explicit injection at the call site
```

L6 makes the dependency *visible*. But `window.Formatter` still has to exist when `chart.js` runs — load order stays implicit.

### Status

| Problem                            | Plain `<script>` | IIFE                       | IIFE + injection                   |
| ---------------------------------- | ---------------- | -------------------------- | ---------------------------------- |
| 1. Name collisions                 | ❌                | ⚠️ mitigated                | ⚠️ mitigated                        |
| 2. Implicit load order             | ❌                | ❌                          | ❌                                  |
| 3. No encapsulation                | ❌                | ✅                          | ✅                                  |
| 4. No explicit dep declaration     | ❌                | ❌                          | ⚠️ partial (visible, not parsable)  |

The IIFE pattern is what every pre-2015 JS codebase looked like — jQuery plugins, Backbone, early Bootstrap JS. It's a remarkable amount of mileage out of just functions and parens. But the wall is structural: **encapsulation is solvable in pure JavaScript; load order and explicit dependencies are not.**

---

## Why a real fix needs the host

The wall in the IIFE pattern points at a specific missing capability. A pure-JS solution to problem 2 would need a function `needs("formatter.js")` that fetches a file, runs it, and makes its API available before the next line. That function can't exist in pure JavaScript — **the language has no fetching primitive**. There's no `download`, no `read`, no network, no filesystem in the language proper. Functions can take arguments, run code, return values, throw, and close over scope. None of those reach outside the JS world.

Every fetch-shaped capability you've ever used in JS — `fetch`, `XMLHttpRequest`, Node's `require()`, `setTimeout`, `console.log`, the DOM — is the **host's** tentacle into JS, exposed as a function-shaped door. The language was designed this way: JS started as a browser-embedded scripting layer where I/O was the embedder's job from day one.

> **Aside —** this is the same engine-vs-runtime distinction surfaced in the asynchronous-JavaScript course. The engine never schedules timers; `setTimeout` is the runtime handing scheduling power to JS through a function. Same shape here — fetching is the host's job, and the language gets `import` syntax (and dynamic `import()`) as the door.

So any real module system has to be a **partnership**:

- **Language side** — per-file scope (so problem 1 + 3 close), syntax to declare exports, syntax to declare dependencies (so problem 4 closes and the host has parseable info).
- **Host side** — a loader that uses the declared dependencies to fetch, link, and order execution (so problem 2 closes).

Neither side alone is enough. This is why `<script>` tags + IIFE were as far as the community could go without spec changes — pure language couldn't reach across files.

---

## Two design axes

Once a real module system is on the table, the design choices cluster on two independent axes.

### Axis 1 — Where does fetching live?

Always in the host. Both CommonJS and ES modules agree on this. The JS engine never opens a file or makes a network request; it asks the host to. Not really a choice — just where the seam sits.

### Axis 2 — When is the dependency graph known?

This is the actual fork.

**Runtime resolution** (CommonJS `require()`, AMD's `require`). Dependencies are discovered by *running the code*. `require()` is a regular function; the argument can be a variable, the call can sit in any branch.

```js
// CommonJS — perfectly legal
if (userPrefersFancy) {                     // L1
  const Fancy = require("./fancy.js");      // L2 — only loaded sometimes
  Fancy.run();                              // L3
}                                           // L4
const lib = require(getLibName());          // L5 — dep name only known at runtime
```

Simple to implement. Works fine when files are local and execution is single-threaded server startup (Node's original use case, 2009).

**Static resolution** (ES modules). Dependencies are discovered by *parsing the syntax*. `import` is a top-of-file declaration with literal string paths — not a function call.

```js
// ESM — only legal at top level, only with literal paths
import { formatCurrency } from "./formatter.js";   // L1
```

The host can scan files without executing them, build the full graph, fetch in parallel, link, then evaluate.

### Decidability — why static unlocks tree-shaking

The deepest consequence of the static-vs-runtime choice is **decidability of static analysis questions** — including, crucially, "is this export ever used?"

With static resolution, a bundler can:

1. **Parse each file without running it.** `import { formatCurrency } from "./formatter.js"` is syntax — no execution needed to extract the dependency name and the imported binding.
2. **Build the full dep graph.** Walk from the entry point, follow every `import`, never execute a line.
3. **Resolve usage per export.** For each `export` in `formatter.js`, ask "does any importer name this binding?" Yes → keep. No → drop.

Step 3 only works because step 2 is complete and trustworthy. Both rest on step 1.

With runtime resolution, every step breaks: `require(someVar)` is legal; `if (cond) { require("./x.js") }` is legal; `module.exports = computeStuff()` makes the exported shape itself the result of running code. To know what's used, the bundler would have to **simulate execution** — branches, conditions, dynamic strings, all of it. That's not "harder static analysis"; it's a different problem class. **Static analysis is decidable; arbitrary code execution simulation is undecidable in general** (halting problem territory). CJS bundlers exist and *try*, but they fall back to "keep everything" whenever they can't prove non-use — which is most non-trivial code.

So the structural reason ES modules enable tree-shaking isn't "tree-shaking happens before code runs." It's: **static syntax makes the dep graph extractable by parsing, which makes export-usage a decidable static-analysis problem.** Runtime resolution turns the same question into "what would this code do?" — and that's not decidable.

---

## CommonJS vs ES modules at a glance

| Aspect                | CommonJS (`require`)                          | ES modules (`import`)                                  |
| --------------------- | --------------------------------------------- | ------------------------------------------------------ |
| Solves problem 1?     | ✅ (each file gets a scope wrapper)            | ✅ (every file is its own scope by language rule)       |
| Solves problem 2?     | ✅ runtime — synchronous fetch                 | ✅ static — host fetches graph before evaluation        |
| Solves problem 3?     | ⚠️ partial (whatever's not on `module.exports`) | ✅ (only `export`-marked bindings escape)               |
| Solves problem 4?     | ⚠️ as a function call, anywhere in the file    | ✅ as a top-of-file declaration, parseable in isolation |
| Dep graph known       | At runtime                                    | At parse time                                          |
| Tree-shakeable        | ❌ in general                                  | ✅                                                      |
| Browser-friendly load | ❌ (sync fetch over network would freeze)      | ✅ (parallel host-side prefetch)                        |
| Top-level `await`     | ❌ (would block the synchronous `require`)     | ✅                                                      |
| Static error catching | ❌ (typos throw at execution)                  | ✅ (parse-time check)                                   |

CommonJS is the simpler design and shipped six years earlier. ES modules pay a complexity cost (a multi-phase loader, live bindings, more nuanced lifecycle) and buy a different *class* of guarantees — guarantees the runtime-resolution approach can't provide regardless of how clever the implementation gets.

---

## What a module system must provide

Pulling the requirements out of everything above. Each requirement is the inverse of a pain point.

1. **Per-file scope.** Each file gets its own scope; top-level declarations don't leak to a shared global. Closes problem 1 and problem 3.
2. **A way to declare exports.** A file says "this is my public API; everything else stays private." Closes problem 3's intentional-API half.
3. **A way to declare dependencies.** A file says "I need *this* binding from *that* file" — closes problem 4. The declaration also gives the loader the info it needs for the next requirement.
4. **A loader that orders execution by dependency.** Fetches files, links them by name, runs them in dep order — closes problem 2. Lives in the host.

### What ES modules deliver

| Requirement                | ESM mechanism                                                    | Where it's covered later                                |
| -------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------- |
| 1. Per-file scope          | Every module file is its own scope by language rule              | *ES module basics*                                      |
| 2. Declare exports         | `export` syntax (named, default, re-exports)                     | *ES module basics*                                      |
| 3. Declare dependencies    | `import` syntax (static), `import()` expression (dynamic)        | *ES module basics*, *Dynamic `import()`*                |
| 4. Loader orders execution | Module Record lifecycle: Parse → Instantiate → Evaluate          | *Static linking & live bindings*, *Module record lifecycle* |

ESM also gives capabilities pure-runtime systems can't match — all flowing from the static-resolution choice on axis 2:

- **Live bindings** — imports are *references* to the exporter's binding, not snapshots. Re-export and circular-dep semantics fall out of this. Covered in *Static linking & live bindings*.
- **Top-level `await`** — possible because the loader knows the full graph and can suspend a module's evaluation step without losing track of dependents. Covered in *Top-level `await`*.
- **Tree-shaking** — bundlers can statically determine which exports are unused. Covered in *Bundlers & the module graph*.

### Static is the default; runtime is an opt-in escape hatch

ESM also ships **dynamic `import("./foo.js")`** — a function-shaped expression that's runtime-resolved, returns a Promise, and is intentionally separate syntax from `import`. The design isn't "static everywhere"; it's "static is the default, runtime is a labeled escape hatch." That separation keeps static guarantees intact for the bulk of code while giving an opt-in door for lazy loading and conditional features. Covered in *Dynamic `import()`*.

---

## The mental hook

When you write `import { formatCurrency } from "./formatter.js"`, you're not calling a function. You're handing the host a parseable declaration. The host fetches, the engine parses and links, and only *then* does any of your code run. That ordering — **parse-and-link before any evaluation** — is the whole game. Every ESM feature with a "wow" answer (live bindings, top-level `await`, tree-shaking, circular-dep handling) is a consequence of it.

---

## Quick reference

- **The pre-module mechanism** — every top-level `var`/`function` in every classic `<script>` becomes a property of the shared global object. One mechanism, four independent pain points fall out.
- **The four pain points** — name collisions, implicit load order, no encapsulation, no explicit dep declaration. Independent: fixing any one alone doesn't fix the others.
- **The IIFE wall** — pure JS gives you function scope (closes encapsulation) but no fetching primitive (can't close load order). Encapsulation is a *language* problem; loading is a *host* problem.
- **The two design axes** — *where fetching lives* (always host, both CJS and ESM) and *when the dep graph is known* (runtime à la `require()` vs static à la `import` — the actual fork).
- **Why static unlocks tree-shaking** — static dep graphs make export-usage a decidable static-analysis problem; runtime dep discovery turns it into general code-simulation, which isn't decidable. The mechanism is decidability, not timing.
- **The mental hook** — `import` is a parseable declaration, not a function call. Parse-and-link before any evaluation is the whole game.
