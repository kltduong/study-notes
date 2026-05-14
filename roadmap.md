# Study Roadmap — Frontend Fundamentals

Goal: whole-system literacy (frontend included) for a backend-strong dev. Reviewing and architecting UI work alongside AI/devs, not specializing.

Last updated: 2026-05-08.

## Path

| #   | Course                     | Status        | Scope                                                                                                                                                      |
| --- | -------------------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `js-vars-scope`            | in progress   | Finish closures + variable lifecycle                                                                                                                       |
| 2   | `js-inheritance`           | in progress   | Finish chains, class syntax, composition (~30% remaining)                                                                                                  |
| 3   | `js-values-fn-this`        | new           | Values & memory model (primitive vs object, slot vs heap reference, copy semantics, identity), Reference type, `this` determination (one rule + overrides), `call`/`apply`/`bind`, arrow functions, `new` protocol, class & `this`, patterns (~8 chunks) |
| 4   | `js-modules`               | new           | Why modules exist, ES module syntax, static linking, live bindings, module record lifecycle, circular deps, dynamic `import()`, Node dual-mode, browser loading, bundlers, top-level `await` (~13 chunks) |
| 5   | DOM fundamentals           | new           | Tree, querying, mutation, event dispatch, delegation, attrs vs props, input, forms, custom events, observers, capstone with CRP-lite (~11 chunks)          |
| 6   | TypeScript fundamentals    | new           | Structural typing, primitives/literals, unions/intersections, generics + inference, narrowing, common utility types, working with libraries (~9–10 chunks) |
| 7   | CSS layout fundamentals    | new           | Box model, normal flow, positioning, stacking contexts, flexbox, grid, responsive, typography essentials (~8–10 chunks)                                    |
| 8   | Browser rendering pipeline | new, optional | Parse → CSSOM → render tree → layout → paint → composite, layer promotion, Core Web Vitals, profiling (~6–8 chunks)                                        |
| 9   | React fundamentals         | new           | Rendering model, components/props/state, derived state vs effects, lists/keys, forms, context, data fetching, composition (~10 chunks)                     |

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

## Order rationale

- **1 → 2** — close open loops; preserve calibration data; promote `shaky` → `solid`.
- **2 → 3** — value semantics (primitive vs reference) are needed to understand what `this` *is* (a reference in a slot); `this` determination builds directly on EC/ER internals from #1; js-inheritance needs `this` in constructors/methods, so cover it first.
- **3 → 4** — modules build on scope (ER types, lexical binding) and `this` (module scope has no global `this`); static linking is easier to reason about with the EC model in memory.
- **4 → 5** — warm cache from `async-js`; substrate-before-abstractions matches formal-reasoning bias. DOM assumes script-vs-module distinction is already clear.
- **5 → 6** — TS leverages reading comprehension on every later course; types land concretely on the DOM APIs just learned.
- **6 → 7** — CSS is the biggest blind spot for backend-strong devs; closes the visible-UI gap.
- **7 → 8** — CRP needs CSSOM literacy to make sense; safely skippable since DOM capstone gives the lite version.
- **8 → 9** — React is the capstone; every prior pillar feeds into understanding why it works the way it does.

## Off-ramps

- **Stop after #7** — ~2.5 months of work, reasonably literate, can review most UI work.
- **Stop after #9 (skipping #8)** — full path minus engine depth.
- **Complete through #9 including #8** — full fundamentals depth; ready to architect, not just review.

## Rough timeline

At one chunk per session and a short session per day: ~3.5–4.5 months end-to-end after #2 closes out. Off-ramping at #6 cuts to ~2.5 months.

## Revising

- Update **Status** as courses progress / complete.
- Adjust scope chunk counts after each course's calibration.
- Move topics from **Off-path** into the main path if a project triggers them.
- Promote artifacts from `notes/` to `courses/` if a notes folder balloons past ~5 entries on the same theme.
- Bump the `Last updated` date when revising.
