# Why Modules Exist

**TL;DR.** Pre-2015 JS had no file-level scope: every `<script>` shared one global Environment Record. This produced three structural failures — namespace pollution, implicit load order, and no encapsulation. The IIFE pattern (Immediately Invoked Function Expression — a function expression run on definition) papers over scope and encapsulation by using closure, but cannot fix load order, because load order is host-level (HTML tag order), not language-level. A real module system must provide five things: per-file scope, explicit exports, explicit imports, a loader/linker, and single instantiation per module.

## The script-tag world

A classic web page in 2014 loads JS via blocking `<script>` tags that execute in document order:

```html
<script src="utils.js"></script>     <!-- L1 -->
<script src="widgets.js"></script>   <!-- L2 -->
<script src="app.js"></script>       <!-- L3 -->
```

There is exactly one Environment Record visible to all of them: the **global ER** (`window` in browsers). Every top-level `var`, `function`, `class`, and (in non-strict scripts) bare assignment becomes a property on it. Files have no scope of their own; tag boundaries are an HTML concern, invisible to the language.

Three failure modes follow directly.

### Failure 1 — namespace pollution

```javascript
// utils.js
var formatDate = function (d) { return d.toISOString(); };       // L1

// widgets.js
var formatDate = function (d) { return d.toLocaleString(); };    // L1
function renderCalendar() { return formatDate(new Date()); }     // L2

// app.js
console.log(formatDate(new Date()));                             // L1
renderCalendar();                                                // L2
```

Both `formatDate(new Date())` (`app.js` L1) and `renderCalendar()` (which calls `formatDate` internally) print the **locale** version. Mechanism, in three steps:

1. **No per-file scope.** Top-level `var formatDate = …` writes to `window.formatDate`. Same for `function` and `class` declarations.
2. **Same-name globals overwrite.** `widgets.js` runs after `utils.js`, so `window.formatDate` is reassigned. The previous binding is gone.
3. **Late-bound name lookup.** `renderCalendar` does *not* capture `formatDate` at definition. Its body says "look up `formatDate` *when called*." The lookup walks the scope chain to the global ER, finds whatever is on `window.formatDate` at that moment.

Consequence: any later script — yours, a vendor's, a browser extension — can rebind `window.formatDate` and silently change `renderCalendar`'s behavior. Author intent has no enforcement.

### Failure 2 — implicit load order

Reorder the same three files so `app.js` loads first:

```html
<script src="app.js"></script>
<script src="utils.js"></script>
<script src="widgets.js"></script>
```

`app.js`'s `formatDate(new Date())` runs before either library has executed. `window.formatDate` doesn't exist yet:

```
Uncaught ReferenceError: formatDate is not defined
```

The error fires at **call time**, not at load time. There is no static check that says "this file depends on `formatDate` from another file; refuse to start until it exists." The dependency is implicit, encoded only in the order humans paste tags into HTML.

This is the load-bearing failure: dependencies between files are **not stated in the source**. They live in HTML.

### Failure 3 — no encapsulation

```javascript
// widgets.js
function pad(n) { return n < 10 ? "0" + n : "" + n; }                      // L1
function formatTime(d) { return pad(d.getHours()) + ":" + pad(d.getMinutes()); } // L2
```

The author intends `pad` as a private helper. The script-tag world has no syntax for that — `function pad` becomes `window.pad`. From `app.js` (or devtools, or any third-party script):

- `pad(7)` works — it's globally accessible.
- `window.pad = () => "💀"` replaces it. Late-bound lookup means `formatTime` then uses the replacement on its next call.

Privacy is unenforceable.

### Common root

All three failures share one cause: **one Environment Record visible to every script.** No file boundary in the language → no scope boundary at file granularity → no privacy, no name isolation, no static dependency tracking.

## The IIFE workaround

Pre-2015, the industry response to **#1** and **#3** was the **IIFE** — Immediately Invoked Function Expression:

```javascript
// widgets.js
var Widgets = (function () {                                       // L1
  function pad(n) { return n < 10 ? "0" + n : "" + n; }            // L2
  function formatTime(d) {                                          // L3
    return pad(d.getHours()) + ":" + pad(d.getMinutes());          // L4
  }                                                                 // L5
  function renderCalendar() { /* uses formatTime */ }               // L6

  return { formatTime: formatTime, renderCalendar: renderCalendar };// L8 — A
})();                                                               // L9
```

### Mechanism

The IIFE creates a **function-scoped Environment Record** — call it the *Widgets ER*. `pad`, `formatTime`, `renderCalendar` are bindings in that ER. Only the global binding `Widgets` (one name on `window`) leaks out, and only the explicitly-returned object on line `L8 — A` is visible through it.

Privacy comes from **closure**, not from a `private` keyword:

- The Widgets ER is not garbage-collected after the IIFE returns — `formatTime` and `renderCalendar` are closures over it. They keep it alive so they can resolve `pad` when called later.
- Outside code has no way to reach into the ER. JavaScript exposes no construct for dumping or enumerating an Environment Record. The only escape hatch is the explicitly-returned object.

So `Widgets.formatTime(new Date())` works. `Widgets.pad(7)` throws — and the precise error matters:

```
TypeError: Widgets.pad is not a function
```

Not a `ReferenceError`. The reasoning chain:

1. The name `Widgets` exists (it's a property on `window`).
2. Property access `Widgets.pad` returns `undefined` — JS doesn't throw on missing properties.
3. Calling `undefined` as a function throws `TypeError`.

> **Aside —** the "property miss returns `undefined`, then call throws TypeError" pattern is the canonical failure mode for "I expected a method but typo'd the name or the object's the wrong shape." Bookmark it.

### What it fixes

| Failure                  | Fixed?                                |
| ------------------------ | ------------------------------------- |
| #1 namespace pollution   | ✅ One global per library               |
| #2 implicit load order   | ❌ Untouched                            |
| #3 no encapsulation      | ✅ Real privacy via closure             |

### What it can't fix

**#2 stays broken**, and it cannot be patched from inside the language. If `app.js` runs before `widgets.js`, `window.Widgets` is `undefined` at the moment `app.js` calls `Widgets.renderCalendar()` — same shape as before, now `TypeError: Cannot read properties of undefined (reading 'renderCalendar')`.

The deeper reason: load order is determined by **HTML tag order**, which is host-level, not language-level. The IIFE is a JS pattern operating on values already in memory. It has no say in whether `widgets.js` has been fetched, parsed, and executed yet. There is no construct in pre-2015 JS where one file can declare *"I depend on `Widgets`; refuse to start until it exists."*

Two adjacent limitations the IIFE pattern also can't address:

- **No static dependency analysis.** Tools can't determine — without running the code — what a file needs. Any "what does this file depend on?" tool has to parse heuristically or execute. Blocks tree-shaking, dead-code elimination, and reliable bundling.
- **No tooling for sharing within a project.** A 30-file app either pollutes `window.MyApp.utils` by hand or duplicates code. There is no `import` to say *"use the `formatTime` from `./time-utils.js`."*

> **Aside —** AMD (RequireJS) and CommonJS (Node's `require`) both grew in this gap, adding a *module loader* on top of the language — runtime-managed dependency graphs, sometimes async (AMD), sometimes sync (CJS). They patched the load-order problem at the cost of a non-standard runtime convention. ES modules later baked this into the language itself, with one critical change: the dependency graph is **static** (knowable before any code runs), not built up dynamically by `require()` calls. That static guarantee is what unlocks tree-shaking and live bindings.

## Requirements for a real module system

Derive the requirements from the three failures plus the IIFE patch's gap:

### 1. Per-file scope

Each file gets its own Environment Record. Top-level `let`, `const`, `function`, `class` bind there, not on the global. Fixes **#1** at the *language* level — no wrapping needed, and *no escape*: even if the author wanted to leak globally, the language refuses.

### 2. Explicit exports

A file declares what it makes available. Anything not exported is private by default. Fixes **#3** with stronger guarantees than IIFEs: no wrapper to remember, no risk of returning the wrong shape, statically inspectable.

### 3. Explicit imports

A file declares what it needs, by name and by source specifier. Combined with #1 and #2, the dependency graph is now **stated in the source**, not implied by HTML tag order. Structural fix to **#2** — "what does this file depend on?" becomes answerable by *reading*, not by *running*.

### 4. A loader / linker

Something has to read those import declarations, fetch the dependencies, and wire bindings together before any user code runs. The module system replaces ad-hoc HTML tag ordering with a real loader that walks the dependency graph automatically.

### 5. Single instantiation per module

If `app.js` and `helpers.js` both import `./logger.js`, the loader must give them *the same* logger module — not run it twice. Otherwise shared state (counters, caches, configuration) fragments. Module-level singletons only work if every importer sees the same instance.

ES modules guarantee this via the **module map** — one record per resolved URL, cached at first import.

> **Aside —** the easy miss when sketching this from scratch. Scope/exports/imports/loader is the obvious set; without single-instantiation the loader gives isolation but no shared identity. Module-level state (config objects, connection pools, registered handlers) silently breaks when the same module loads twice.

### Forward-looking

Two refinements that matter for later notes:

- **"Static" imports are stronger than "declared in syntax."** ES modules enforce that `import` / `export` are only legal at the top level of a module — not inside `if`, not inside functions. This makes the graph parseable *before any code runs*. CommonJS's `require()` is a function call, so its dependencies emerge only at runtime. The static-vs-dynamic distinction unlocks tree-shaking and live bindings — covered in *Static linking & live bindings*.
- **The loader is host-level.** The engine parses and links; the *host* (Node, browser) supplies the loader that resolves a specifier like `"./helpers.js"` or `"lodash"` to actual bytes. Engine-vs-runtime split — covered in *Module record lifecycle*.

## Quick reference

- **Root cause of pre-module pain** — one global Environment Record visible to every script; no file-level scope.
- **Three failures** — namespace pollution (#1), implicit load order (#2), no encapsulation (#3).
- **IIFE coverage** — fixes #1 and #3 via closure; cannot fix #2 because load order is host-level (HTML tag order).
- **Late-bound lookup** — function bodies resolve free names at call time against the scope chain; reassigning a global silently changes any function that references it.
- **Property-miss failure shape** — `obj.missing` returns `undefined` (no error); calling `undefined(...)` throws `TypeError`. Not a `ReferenceError`.
- **Five module-system requirements** — per-file scope, explicit exports, explicit imports, a loader/linker, single instantiation per module.
