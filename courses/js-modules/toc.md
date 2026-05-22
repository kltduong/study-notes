# JS Modules — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

_(To be filled at session start.)_

## Progress

- [ ] **Why modules exist** — the pre-module problem (global namespace pollution, implicit load order, no encapsulation), IIFE workaround and its limits, what a module system needs to provide
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
