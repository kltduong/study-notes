# Why Modules Exist — Teaching Draft

## Plan (teaching order)

- [x] **Teaser** — multi-script page, predict the bug
- [x] **Pre-module pain points** — three concrete failures of the script-tag world
- [x] **The IIFE workaround** — what it fixed, what it couldn't
- [x] **What a module system must provide** — derive from the failures above
- [ ] **Sub-part check** (rolled into end-of-chunk understanding check)

---

## Teaser

You have a static HTML page in 2014. Two libraries, then your app code:

```html
<script src="https://cdn.example.com/utils.js"></script>  <!-- L1 -->
<script src="https://cdn.example.com/widgets.js"></script> <!-- L2 -->
<script src="./app.js"></script>                            <!-- L3 -->
```

Each file is just a sequence of statements that runs top-to-bottom in the browser. No `import`, no `export`, no bundler. They all share one global object (`window`).

`utils.js` defines:

```javascript
// utils.js
var formatDate = function (d) { return d.toISOString(); };  // L1
```

`widgets.js` (written by a different team / vendor) defines:

```javascript
// widgets.js
var formatDate = function (d) { return d.toLocaleString(); }; // L1
function renderCalendar() { /* uses formatDate internally */ } // L2
```

`app.js` does:

```javascript
// app.js
console.log(formatDate(new Date())); // L1
renderCalendar();                    // L2
```

**Predict:** what does the page print on `L1` of `app.js`, and what does `renderCalendar()` use internally? Why?



## Reveal

Both call sites use `widgets.js`'s `formatDate`. The locale version wins.

The right *mechanism* for this — not just the conclusion:

1. **No per-file scope.** A classic `<script>`'s top level *is* the global scope. `var formatDate = …` and `function pad(…)` both create properties on the global object (`window` in browsers).
2. **Same-name globals overwrite.** When `widgets.js` runs after `utils.js`, `window.formatDate` is reassigned. The previous binding is gone.
3. **Late-bound name lookup.** `renderCalendar` does *not* capture `formatDate` at definition time. Its body says "look up the name `formatDate` *when called*." The lookup walks the scope chain, finds `window.formatDate` at call time. Reassign the global later → `renderCalendar()` silently switches.

> **Aside —** this is why `renderCalendar()` and `app.js`'s direct call agree: both resolve `formatDate` at call time, against the same global record, after both library scripts have run.

## Pre-module pain points

The three failure modes the script-tag world ships with.

### Global namespace pollution

Already shown by the teaser. Every top-level `var` and `function` becomes a property on `window`. Two scripts that pick the same name silently clobber each other. Late-bound lookup means even functions that look "internal" (`renderCalendar`) snap to whichever version of the global exists at call time.

There is no syntax in classic scripts for "this name is private to this file." Files don't have scopes — only the global scope does.

### Implicit load order

Reorder the tags by mistake:

```html
<script src="./app.js"></script>                            <!-- L1 -->
<script src="https://cdn.example.com/utils.js"></script>    <!-- L2 -->
<script src="https://cdn.example.com/widgets.js"></script>  <!-- L3 -->
```

When `app.js`'s `formatDate(new Date())` runs, neither `utils.js` nor `widgets.js` has executed yet (classic `<script>` tags are blocking and execute in document order). Lookup of `formatDate` fails:

```
Uncaught ReferenceError: formatDate is not defined
```

The error fires **at call time on `app.js` line 1** — not at parse time, not at load time. There is no static check that says "you depend on `formatDate` from another file, that file isn't loaded yet, refuse to start." The dependency is implicit in the tag order. Tag order is maintained by humans, by hand, in HTML — and breaks silently when reshuffled.

### No encapsulation

```javascript
// widgets.js
function pad(n) { return n < 10 ? "0" + n : "" + n; }                  // L1
function formatTime(d) { return pad(d.getHours()) + ":" + pad(d.getMinutes()); } // L2
function renderCalendar() { /* … */ }                                  // L3
```

The author intends `pad` as a private helper for `formatTime`. The script-tag world has no way to express that. `function pad(…)` at the top of a classic script becomes `window.pad`. From `app.js` (or devtools, or a third-party script, or a malicious extension):

- `pad(7)` works.
- `window.pad = () => "💀"` replaces it; `formatTime` then uses the replacement at its next call (late-bound lookup again).

Author's *intent* has no enforcement. Every top-level name is part of the public API of the page.

> **Aside —** all three failure modes share a root cause: there is exactly one Environment Record visible to every script. No file boundary in the language → no scope boundary at file granularity → no privacy, no name isolation, no static dependency tracking.



## The IIFE workaround

Pre-2015, the industry response to **#1 (namespace pollution)** and **#3 (no encapsulation)** was the **IIFE** — Immediately Invoked Function Expression. A function expression that runs the moment it's defined, returning the value the rest of the world should see.

```javascript
// widgets.js
var Widgets = (function () {                                  // L1
  function pad(n) { return n < 10 ? "0" + n : "" + n; }       // L2
  function formatTime(d) {                                     // L3
    return pad(d.getHours()) + ":" + pad(d.getMinutes());     // L4
  }                                                            // L5
  function renderCalendar() { /* uses formatTime */ }          // L6

  return { formatTime: formatTime, renderCalendar: renderCalendar }; // L8 — A
})();                                                          // L9
```

### Mechanism

The IIFE creates a **function-scoped Environment Record** — call it the *Widgets ER*. `pad`, `formatTime`, `renderCalendar` are all bindings in that ER. Only the global binding `Widgets` (one name on `window`) leaks out, and only the explicitly-returned object (line `L8 — A`) is visible through it.

Privacy comes from **closure**, not from a `private` keyword. After the IIFE returns:

- The Widgets ER is *not* garbage-collected — `formatTime` and `renderCalendar` are closures over it. They need it to resolve `pad` when called later.
- Outside code has no way to reach into the ER. JavaScript exposes no construct for dumping or enumerating an Environment Record. The only escape hatch is the explicitly-returned object.

So `Widgets.formatTime(new Date())` works; `Widgets.pad(7)` throws — but the precise error matters:

```
TypeError: Widgets.pad is not a function
```

Not a `ReferenceError`. The reasoning chain:

1. The name `Widgets` exists (it's a property on `window`).
2. Property access `Widgets.pad` doesn't fail — JS returns `undefined` for missing properties. No reference error.
3. Calling `undefined` as a function throws `TypeError`.

> **Aside —** the "property miss returns `undefined`, then call throws TypeError" pattern shows up everywhere in JS. Bookmark it as the canonical failure mode for "I expected a method but typo'd the name or the object's the wrong shape."

### What IIFEs fix

- **#1 namespace pollution** — one global (`Widgets`) instead of one per top-level name.
- **#3 encapsulation** — privacy via closure. Real, language-enforced, not by convention.

### What IIFEs cannot fix

**#2 implicit load order** is untouched, and it cannot be touched from *inside* the language. If you reorder tags so `app.js` runs before `widgets.js`, then `Widgets` is `undefined` on `window` at the moment `app.js` calls `Widgets.renderCalendar()` — same failure shape as before, now `TypeError: Cannot read properties of undefined (reading 'renderCalendar')`.

The deeper reason: load order is determined by **HTML tag order**, which is host-level, not language-level. The IIFE is a JS pattern operating on values that already exist in memory. It has no say in whether `widgets.js` has been fetched, parsed, and executed yet. There is no construct in pre-2015 JS where one file can declare *"I depend on `Widgets`; refuse to start until it exists."*

Two adjacent limitations the IIFE pattern also can't address — both motivate later module features:

- **No static analysis of dependencies.** Tools can't tell — without running the code — what a file needs. Any "what does this file depend on?" tool has to parse heuristically or execute. This blocks tree-shaking, dead-code elimination, and reliable bundling.
- **No tooling for sharing within a project.** A 30-file app either pollutes `window.MyApp.utils` by hand or duplicates code. There's no `import` to say "use the `formatTime` from `./time-utils.js`."

> **Aside —** AMD (RequireJS) and CommonJS (Node's `require`) both grew up in this gap. They added a *module loader* on top of the language — runtime-managed dependency graphs, sometimes async (AMD), sometimes sync (CJS). They patched the load-order problem at the cost of a non-standard runtime convention. ES modules later baked this into the language itself, but with a key change: the dependency graph is **static** (knowable before any code runs), not built up dynamically by `require()` calls. That static guarantee is what unlocks tree-shaking and live bindings — covered in later chunks.



## What a module system must provide

Derive the requirements from the three pain points (`#1`, `#2`, `#3`) plus the IIFE patch's gap.

### 1. Per-file scope

Each file gets its own Environment Record. Top-level `let`, `const`, `function`, `class` bind there, not on the global. Fixes **#1** at the *language* level — no IIFE wrapping needed, and *no escape*: even if a module author wanted to leak globally, the language refuses to put top-level bindings on `window`.

### 2. Explicit exports

A file declares what it makes available. Anything not exported is private by default — privacy is the default, leakage requires opt-in. Fixes **#3** with stronger guarantees than IIFEs:

- No wrapper to remember.
- No risk of accidentally returning the wrong shape.
- Static — the export list is part of the file's parseable structure.

### 3. Explicit imports

A file declares what it needs, by name and by source specifier. Combined with #1 and #2, the dependency graph is now **stated in the source**, not implied by HTML tag order. This is the structural fix to **#2** — the system can now answer "what does this file depend on?" by *reading*, not by *running*.

### 4. A loader / linker

Something has to read those import declarations, fetch the dependencies, and wire bindings together before any user code runs. Imports are useless if the system can't act on them. The module system replaces ad-hoc tag ordering with a real loader that resolves the dependency graph automatically.

### 5. Single instantiation per module

If `app.js` and `helpers.js` both import `./logger.js`, the loader must give them *the same* logger module — not run it twice. Otherwise shared state (counters, caches, configuration) fragments. Module-level state only works if every importer sees the same instance.

ES modules guarantee this via the **module map** — one record per resolved URL. Cached at first import, reused thereafter.

> **Aside —** the easy-to-miss requirement. Scope/exports/imports/loader is the obvious set; without single-instantiation the loader gives isolation but no shared identity. Module-level singletons (config object, connection pool, registered handlers) silently break when the same module is loaded twice.

### Forward-looking

Two refinements that matter for later chunks:

- **"Static" imports — a stronger property than "declared in syntax."** ESM enforces that `import` / `export` are only legal at the top level of a module — not inside `if`, not inside functions. This makes the dependency graph parseable *before any code runs*. CommonJS's `require()` is a function call, so its dependencies emerge only at runtime. The static-vs-dynamic distinction is what unlocks tree-shaking and live bindings — see *Static linking & live bindings*.
- **The loader is host-level.** The engine parses and links; the *host* (Node, browser) supplies the loader that resolves a specifier like `"./helpers.js"` or `"lodash"` to actual bytes. Engine-vs-runtime split — see *Module record lifecycle*.

