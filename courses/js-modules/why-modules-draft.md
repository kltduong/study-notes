# Why modules exist — draft

Teaching draft for the chunk. Lives alongside chat; final note is reorganized after the chunk gate.

## Plan (teaching order)

- [x] **Motivation** — implementation challenge: build a multi-file dashboard with no `import`/`export`, no bundler, just `<script>` tags. Feel the pain.
- [x] **The pre-module landscape** — *folded into the motivation reveal:* the single-shared-global mechanism + four pain points (name collisions, implicit load order, no encapsulation, no explicit dependency declaration), with the table showing they're independent.
- [ ] **IIFE namespace pattern** — the pre-ES2015 workaround. What it fixes (encapsulation, single global per module). What it can't fix (still need `<script>` ordering, deps still implicit, still leaks one global per file).
- [ ] **What a module system needs to provide** — the requirements list extracted from the pain points, framed as four things ES modules deliver on. Bridge to the next chunk (ES module syntax).

---

## Motivation — implementation challenge

> You're building a small dashboard. Three logical pieces:
>
> - `formatter.js` — exposes two helpers: `formatCurrency(n)` and `formatDate(d)`.
> - `chart.js` — exposes one helper: `renderChart(data, container)`. Internally it calls `formatCurrency` to format axis labels.
> - `app.js` — entry point. Calls `renderChart(...)` to draw a chart into a `<div>`.
>
> **Constraint:** pretend it's 2010. No `import`/`export`. No `require`. No bundler. Just `index.html` with plain `<script>` tags.
>
> Sketch:
>
> 1. The `<script>` tags in `index.html`.
> 2. How `formatter.js` makes its two functions reachable from `chart.js`.
> 3. How `chart.js` makes `renderChart` reachable from `app.js`, and how it gets at `formatCurrency`.
> 4. How `app.js` calls `renderChart`.
>
> Skip the actual implementations — just the *shape* of how each file exposes its API and how the others reach it. Aim for something a dev in 2010 would actually have shipped.

Once you've sketched it, we'll walk through what's painful about it, and that pain becomes the motivation for the rest of the chunk.


### Your sketch (concrete form)

```html
<!-- index.html -->
<script src="formatter.js"></script>  <!-- L1 -->
<script src="chart.js"></script>      <!-- L2 -->
<script src="app.js"></script>        <!-- L3 -->
```

```js
// formatter.js
function formatCurrency(n) { /* ... */ }   // L1
function formatDate(d)     { /* ... */ }   // L2
```

```js
// chart.js
function renderChart(data, container) {    // L1
  const label = formatCurrency(data[0]);   // L2 — reaches into formatter.js's stuff
  // ...                                   // L3
}                                          // L4
```

```js
// app.js
renderChart(myData, document.querySelector("#chart"));  // L1
```

This is exactly what a 2010 dev would have shipped — and it works. The mechanism that makes it work is also the source of every problem.

### The reveal — one mechanism, four problems

**The mechanism.** Plain `<script>` tags execute their files top-to-bottom into **one shared global scope** (`window` in the browser). A `function foo() {}` at top level in `formatter.js` becomes `window.foo`. When `chart.js` later writes `formatCurrency(...)`, the lookup walks the scope chain, finds nothing local, and lands on `window.formatCurrency` — the same global the previous script wrote.

> **Aside —** this is the "single shared global" model from `js-vars-scope`. Every top-level `function`/`var` declaration in any script becomes a property of the same global object. Nothing in the language treats files as separate units.

That single mechanism (one shared global) creates four distinct problems. Each one is genuinely independent — solving any one of them doesn't solve the others.

#### Problem 1 — Name collisions

Two files declaring the same name silently overwrite each other.

```js
// formatter.js
function format(n) { return `$${n}`; }     // L1

// chart.js
function format(d) { return d.toFixed(2); } // L1 — same global slot, formatter's version is gone
```

No error. No warning. The second file wins because the second `<script>` runs later. As soon as you pull in a third-party library, this gets unmanageable — every helper they expose is yours to step on, and vice versa.

#### Problem 2 — Implicit load order

The HTML tag order *is* the dependency declaration. `chart.js` works only because `formatter.js` ran first and parked `formatCurrency` on `window`. Reorder the tags:

```html
<script src="chart.js"></script>      <!-- L1 -->
<script src="formatter.js"></script>  <!-- L2 -->
<script src="app.js"></script>        <!-- L3 -->
```

`chart.js` parses fine — function declarations don't call anything yet. But the moment `app.js` calls `renderChart(...)`, the body of `renderChart` executes, hits `formatCurrency(...)`, and... actually it still works here, because by L3 `formatter.js` has already run. Try the *real* failure case:

```html
<script src="chart.js"></script>      <!-- L1 -->
<script>renderChart(myData, ...);</script>  <!-- L2 — runs before formatter.js -->
<script src="formatter.js"></script>  <!-- L3 -->
```

Now L2 calls `renderChart` → which calls `formatCurrency` → which is `undefined` → `TypeError: formatCurrency is not a function`. The dependency `chart.js → formatter.js` exists in the code but is invisible to the loader. The HTML author has to know it.

> ⚠️ Subtler failure: a top-level statement in `chart.js` (not inside a function) that *uses* `formatCurrency` at load time would break the moment `chart.js` is placed before `formatter.js` — no waiting for `app.js` to call it. Function declarations are forgiving because they defer the lookup until call time; top-level statements aren't.

#### Problem 3 — No encapsulation

Every top-level name in every file is public. `chart.js` might want a private helper `_pickAxisScale(data)` — there's no way to mark it as internal. Drop it at top level and the whole world (including future files written by your teammate) can see it and call it.

```js
// chart.js
function _pickAxisScale(data) { /* internal! */ } // L1 — actually window._pickAxisScale, fully public
function renderChart(data, container) {            // L2
  const scale = _pickAxisScale(data);             // L3
  // ...                                          // L4
}                                                  // L5
```

The `_` prefix is a *convention*, not a barrier. Anyone can call `_pickAxisScale` from anywhere; refactoring it is now a breaking change.

#### Problem 4 — No explicit dependency declaration

Reading `chart.js` in isolation, what does it depend on? You can't tell. `formatCurrency` is a free reference — it could come from `formatter.js`, from a third-party script, from a global polyfill, from an inline `<script>` block in the HTML. The only way to find out is to grep the entire codebase for the name, or run it and see what breaks.

This is the worst one for large codebases: dependencies are *invisible*, so refactoring is dangerous (you can't know what depends on what), code review can't catch mismatches, and tooling can't help (a static analyzer reading `chart.js` alone has no idea `formatCurrency` is supposed to come from `formatter.js`).

### The four problems are independent

| Problem                        | Symptom                                       | What "fixing" it alone would require                          |
| ------------------------------ | --------------------------------------------- | ------------------------------------------------------------- |
| Name collisions                | Silent overwrite, last-script-wins            | Per-file scope (file is not part of global)                   |
| Implicit load order            | TypeErrors when scripts ordered wrong         | A way for code to *declare* what it needs, loader resolves it |
| No encapsulation               | Internal helpers leak as public globals       | Per-file scope + an opt-in "this is the public API" mechanism |
| No explicit dependency declaration | Free references everywhere, no static signal | Syntax that names which file each external comes from         |

Notice the overlap: per-file scope alone fixes 1 and partially fixes 3. Explicit `import` syntax fixes 4 and gives the loader enough info to fix 2. So a real solution needs **two things together** — per-file scope *and* declarative import/export. Either alone is half a fix.

This is the design target ES modules hit. The next sub-part looks at the partial workaround the community shipped *before* ES modules existed (the IIFE namespace pattern), to see how close you can get with just functions and conventions — and where the wall is.
