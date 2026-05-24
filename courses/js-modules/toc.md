# JS Modules — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

**Starting point.** Working JS dev, ships real code with `import` / `export` (named, default, re-exports). Day-to-day fluency assumed. Fuzzy on:

- Spec-level mechanics — Parse / Instantiate / Evaluate phases.
- *Why* bindings are live, not copied (vs CommonJS `require`).
- How the linker handles circular dependencies.
- What `<script type="module">` actually changes (defer, CORS, strict mode, scope).
- CJS ↔ ESM interop edges in Node.

**Goal.** Clean mental model + implementation fluency. Precise grip on:

- Static-vs-dynamic distinction (when does the engine know the graph?).
- Engine vs host (loader) split — what runs where.
- Environment differences (Node, browser, bundlers) and *why* they differ.

**Planned arc** (3 phases mirroring the chunk groups):

1. **Static model** — pre-module pain → ES module syntax → static linking + live bindings → module record lifecycle. *Quiz.*
2. **Runtime & environment** — circular deps → dynamic `import()` → Node dual-mode → browser loading. *Quiz.*
3. **Production concerns** — bundlers + tree-shaking → top-level `await` → patterns & design. *Final test.*

Order rationale: static first (whole graph known at parse), dynamic second (runtime decisions on top of the static base), tooling third (how real codebases assemble it all).

**Cross-cutting weaknesses to probe** (from learner profile):

- *Engine vs runtime conflation* — loader is host, parse/link/eval is engine. Module map sits at the boundary.
- *Right conclusion, wrong mechanism* — e.g. "imports are hoisted" feels right but the precise mechanism is that linking precedes any evaluation.

## Progress

- [] **Why modules exist** — the pre-module problem (global namespace pollution, implicit load order, no encapsulation), IIFE workaround and its limits, what a module system needs to provide
- [ ] **ES module basics** — `export` / `import` syntax, named vs default exports, re-exports, `export * from`, the module scope (each file is its own Module ER)
- [ ] **Static linking & live bindings** — how the engine resolves imports at parse time (not runtime), why bindings are live (not copied values), contrast with CommonJS `require` copy semantics
- [ ] **Module record lifecycle** — Parse → Instantiate (link) → Evaluate, what happens at each phase, why top-level code runs exactly once, the module map / registry
- [ ] **Quiz: static model** — covers *Why modules exist* through *Module record lifecycle*
- [ ] **Circular dependencies** — how the linker handles cycles, why live bindings make cycles workable (vs CJS), the "temporal hole" problem when a cycle reads before the binding is initialized
- [ ] **Dynamic `import()`** — lazy loading, returns a Promise, use cases (code splitting, conditional loading), contrast with static `import`
- [ ] **`.mjs` vs `.cjs` vs `"type": "module"`** — Node's dual-mode system, file extension rules, `package.json` `"type"` field, interop constraints (CJS can't `import` ESM synchronously)
- [ ] **Browser loading** — `<script type="module">` behavior, deferred by default, CORS requirements, import maps, bare specifiers
- [ ] **Quiz: runtime & environment** — covers *Circular dependencies* through *Browser loading*
- [ ] **Bundlers & the module graph** — what bundlers do (resolve → bundle → tree-shake), why static `import` enables tree-shaking but `require()` doesn't, source maps, the dev vs prod pipeline
- [ ] **Top-level `await`** — how it works in ESM (blocks the parent module's Evaluate phase), why it's only valid in modules, practical patterns and pitfalls
- [ ] **Module patterns & design** — barrel files and their tree-shaking cost, re-export strategies, avoiding circular dep traps in large codebases
- [ ] **Final test** — cumulative across all chunks
