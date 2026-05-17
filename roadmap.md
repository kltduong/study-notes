# Study Roadmap — Frontend Fundamentals

Goal: whole-system literacy (frontend included) for a backend-strong dev. Reviewing and architecting UI work alongside AI/devs, not specializing.

Last updated: 2026-05-17.

## Path

| #   | Course                     | Status        | Scope                                                                                                                                                      |
| --- | -------------------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `js-vars-scope`            | in progress   | Finish closures + variable lifecycle                                                                                                                       |
| 2   | `js-inheritance`           | in progress   | Finish chains, class syntax, composition (~30% remaining)                                                                                                  |
| 3   | `js-values-fn-this`        | new           | Values & memory model (primitive vs object, slot vs heap reference, copy semantics, identity), Reference type, `this` determination (one rule + overrides), `call`/`apply`/`bind`, arrow functions, `new` protocol, class & `this`, patterns (~8 chunks) |
| 4   | `js-modules`               | new           | Why modules exist, ES module syntax, static linking, live bindings, module record lifecycle, circular deps, dynamic `import()`, Node dual-mode, browser loading, bundlers, top-level `await` (~13 chunks) |
| 5   | `js-modern-syntax`         | new           | Syntax fluency — class, fields, private `#`, static, extends/super, destructuring, spread/rest, `?.`, `??`, template literals, iterators, generators, symbols, proxy/reflect, misc modern features (~12 chunks) |
| 6   | DOM fundamentals           | new           | Tree, querying, mutation, event dispatch, delegation, attrs vs props, input, forms, custom events, observers, capstone with CRP-lite (~11 chunks)          |
| 7   | TypeScript fundamentals    | new           | Structural typing, primitives/literals, unions/intersections, generics + inference, narrowing, common utility types, working with libraries (~9–10 chunks) |
| 8   | CSS layout fundamentals    | new           | Box model, normal flow, positioning, stacking contexts, flexbox, grid, responsive, typography essentials (~8–10 chunks)                                    |
| 9   | Browser rendering pipeline | new, optional | Parse → CSSOM → render tree → layout → paint → composite, layer promotion, Core Web Vitals, profiling (~6–8 chunks)                                        |
| 10  | Datastar fundamentals      | new           | Hypermedia-driven reactivity, signals & `data-*` attributes, SSE-based backend push, expressions & events, morphing, forms & CRUD, composition with fragments (~8–10 chunks) |

## Parallel artifact (notes, not a course)

`notes/client-http/` — built lazily as topics arise:

- `fetch` semantics + the `.ok` trap
- `AbortController` + cancellation
- Same-origin policy + CORS
- Streaming (`ReadableStream`, SSE)
- Real-time (WebSockets, decision table vs SSE/polling)

## Off-path — skip unless triggered

- **Web workers** — only if in-browser AI inference / heavy client compute appears.
- **Browser security model course** (CORS + cookies + CSP + iframe + `postMessage` + SRI) — only if a project demands; otherwise CORS lives in `notes/client-http/`.
- **Functions deep-dive (`this`/bind/composition)** — promoted to main path as #3 (`js-values-fn-this`).
- **React fundamentals** — promote to main path only if a project mandates it (client / employer / OSS integration), or if a UI-interactive carve-out appears that Datastar doesn't fit. Estimate: 1–2 week focused sprint, not a full course, given the fundamentals base from #1–#8. Vue not on the table — pivot only if a specific project demands it.

## Order rationale

- **1 → 2** — close open loops; preserve calibration data; promote `shaky` → `solid`.
- **2 → 3** — value semantics (primitive vs reference) are needed to understand what `this` *is* (a reference in a slot); `this` determination builds directly on EC/ER internals from #1; js-inheritance needs `this` in constructors/methods, so cover it first.
- **3 → 4** — modules build on scope (ER types, lexical binding) and `this` (module scope has no global `this`); static linking is easier to reason about with the EC model in memory.
- **4 → 5** — modules introduce import/export syntax; modern-syntax course systematizes the rest of ES2015+ grammar so everything after reads fluently.
- **5 → 6** — DOM code uses classes, destructuring, optional chaining, iterators constantly; syntax fluency first means DOM examples need no syntax detours.
- **6 → 7** — TS leverages reading comprehension on every later course; types land concretely on the DOM APIs just learned.
- **7 → 8** — CSS is the biggest blind spot for backend-strong devs; closes the visible-UI gap.
- **8 → 9** — CRP needs CSSOM literacy to make sense; safely skippable since DOM capstone gives the lite version.
- **9 → 10** — Datastar is the capstone; DOM events, SSE (from `notes/client-http/`), and CSS layout all feed directly into understanding hypermedia-driven reactivity without a virtual DOM layer in between.

## Off-ramps

- **Stop after #8** — ~3 months of work, reasonably literate, can review most UI work.
- **Stop after #10 (skipping #9)** — full path minus engine depth; hypermedia-driven UI covered.
- **Complete through #10 including #9** — full fundamentals depth; ready to architect, not just review.

## Rough timeline

At one chunk per session and a short session per day: ~4–5 months end-to-end after #2 closes out. Off-ramping at #7 cuts to ~3 months.

## Revising

- Update **Status** as courses progress / complete.
- Adjust scope chunk counts after each course's calibration.
- Move topics from **Off-path** into the main path if a project triggers them.
- Promote artifacts from `notes/` to `courses/` if a notes folder balloons past ~5 entries on the same theme.
- Bump the `Last updated` date when revising.
