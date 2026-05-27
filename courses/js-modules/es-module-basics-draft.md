# ES module basics — draft

Teaching draft for the chunk. Lives alongside chat; final note is reorganized after the chunk gate.

## Plan (teaching order)

- [ ] **Motivation (predict-the-failure)** — three-line snippet: top-level `var`, top-level `function`, top-level `this` reference. Predict output. The reveal: every module file is its own scope by language rule, and `this` at the top level is `undefined` (not `globalThis`).
- [ ] **The Module Environment Record** — what "every file is its own scope" actually means: each module gets a fresh Module ER (a flavor of Lexical ER, parent is null), strict-mode-on automatically, top-level declarations are local to it. Contrast with the script-mode global-object scope from the previous chunk.
- [ ] **`export` — named, default, list, renamed** — the four shapes: inline `export const x`, `export function f()`, `export { a, b as c }`, `export default <expr>`. What "default" actually is at the binding level (a binding named `default`, not magic). The right tool when.
- [ ] **`import` — named, default, namespace, side-effect** — the four shapes mirroring exports: `import { a, b as c }`, `import x from`, `import * as ns`, `import "./side-effect.js"`. What each binds in the local Module ER.
- [ ] **Re-exports** — `export { x } from "./y.js"`, `export * from "./y.js"`, `export { default as something } from "./y.js"`. Why this is *not* "import-then-export": the binding is forwarded directly, the re-exporting module never names it locally.
- [ ] **Sub-part check rolled up at the end of the chunk only** (each sub-part still gets its own focused check).

---

## Motivation — predict the failure

You write `import` / `export` daily; the syntax is muscle memory. But before we drill the syntax, here's a snippet that uses *zero* import/export — just three top-level statements in a `.mjs` file. Read it carefully and predict each labelled output.

```js
// puzzle.mjs — run in Node as: node puzzle.mjs
var greeting = "hi";                       // L1
function help() { return 42; }             // L2

console.log("a:", globalThis.greeting);    // L3
console.log("b:", globalThis.help);        // L4
console.log("c:", this);                   // L5
```

For each output `a`, `b`, `c`:

1. What does it print?
2. Why?

If your prediction matches all three and you can articulate *why* in one sentence per line, the rest of the chunk will be syntax recap and you can speed through. If any line surprises you, the underlying rule is the foundation we'll build the rest of the chunk on.

(The reveal is below — try the prediction first.)

---

<!-- to be filled after user predicts -->
