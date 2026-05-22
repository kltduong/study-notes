# Study Roadmap — Frontend Fundamentals

Goal: whole-system literacy (frontend included) for a backend-strong dev. Reviewing and architecting UI work alongside AI/devs, not specializing.

Last updated: 2026-05-22.

## Path

| #   | Course                     | Status        | Progress | Scope                                                                                                                                                      |
| --- | -------------------------- | ------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `js-vars-scope`            | in progress   | 92%      | Finish closures + variable lifecycle                                                                                                                       |
| 2   | `js-inheritance`           | in progress   | 73%      | Finish chains, class syntax, composition (~30% remaining)                                                                                                  |
| 3   | `js-values-fn-this`        | in progress   | 73%      | Values & memory model (primitive vs object, slot vs heap reference, copy semantics, identity), Reference type, `this` determination (one rule + overrides), `call`/`apply`/`bind`, arrow functions, `new` protocol, class & `this`, patterns (~8 chunks) |
| 4   | `js-hof-functional`        | in progress   | 13%      | Functions as values, map/filter/reduce as folds, reduce deep dive, composition & pipelines, currying & partial application, immutability patterns, algebraic structure (monoids, functors), real-world patterns, FP vs OOP tradeoffs (~9 chunks) |
| 5   | `js-modules`               | new           | 0%       | Why modules exist, ES module syntax, static linking, live bindings, module record lifecycle, circular deps, dynamic `import()`, Node dual-mode, browser loading, bundlers, top-level `await` (~13 chunks) |
| 6   | `js-modern-syntax`         | new           | 0%       | Syntax fluency — class, fields, private `#`, static, extends/super, destructuring, spread/rest, `?.`, `??`, template literals, iterators, generators, symbols, proxy/reflect, misc modern features (~12 chunks) |
| 7   | `js-error-handling`        | new           | 0%       | Error anatomy, try/catch/finally semantics, custom errors & cause chaining, propagation strategies, async error handling, validation & defensive input, graceful degradation, patterns & anti-patterns (~8 chunks) |
| 8   | `js-iterators-generators`  | new           | 0%       | Iterator protocol, custom iterables, generator fundamentals, two-way channels (yield as expression), `yield*` & composition, async iterators & `for await...of`, async generators, streams & integration (~8 chunks) |
| 9   | DOM fundamentals           | new           | 0%       | Tree, querying, mutation, event dispatch, delegation, attrs vs props, input, forms, custom events, observers, capstone with CRP-lite (~11 chunks)          |
| 10  | TypeScript fundamentals    | new           | 0%       | Structural typing, primitives/literals, unions/intersections, generics + inference, narrowing, common utility types, working with libraries (~9–10 chunks) |
| 11  | CSS layout fundamentals    | new           | 0%       | Box model, normal flow, positioning, stacking contexts, flexbox, grid, responsive, typography essentials (~8–10 chunks)                                    |
| 12  | Browser rendering pipeline | new, optional | 0%       | Parse → CSSOM → render tree → layout → paint → composite, layer promotion, Core Web Vitals, profiling (~6–8 chunks)                                        |
| 13  | Datastar fundamentals      | new           | 0%       | Hypermedia-driven reactivity, signals & `data-*` attributes, SSE-based backend push, expressions & events, morphing, forms & CRUD, composition with fragments (~8–10 chunks) |

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
- **Functions deep-dive (`this`/bind/composition)** — promoted to main path as `js-values-fn-this`. HoF/functional patterns split out as `js-hof-functional`.
- **React fundamentals** — promote to main path only if a project mandates it (client / employer / OSS integration), or if a UI-interactive carve-out appears that Datastar doesn't fit. Estimate: 1–2 week focused sprint, not a full course, given the fundamentals base from `js-vars-scope` through `js-iterators-generators`. Vue not on the table — pivot only if a specific project demands it.

## Order rationale

- **`js-vars-scope` → `js-inheritance`** — close open loops; preserve calibration data; promote `shaky` → `solid`.
- **`js-inheritance` → `js-values-fn-this`** — value semantics (primitive vs reference) are needed to understand what `this` *is* (a reference in a slot); `this` determination builds directly on EC/ER internals from `js-vars-scope`; js-inheritance needs `this` in constructors/methods, so cover it first.
- **`js-values-fn-this` → `js-hof-functional`** — HoF patterns build directly on closures (from `js-vars-scope`), `this`/arrows/bind (from `js-values-fn-this`), and the value-vs-reference model (from `js-values-fn-this`). Composition and currying need those settled. Also: reduce-as-fold connects to the formal-abstraction preference in the learner profile.
- **`js-hof-functional` → `js-modules`** — modules build on scope (ER types, lexical binding) and `this` (module scope has no global `this`); static linking is easier to reason about with the EC model in memory. HoF patterns (composition, re-exports) make module design choices intuitive.
- **`js-modules` → `js-modern-syntax`** — modules introduce import/export syntax; modern-syntax course systematizes the rest of ES2015+ grammar so everything after reads fluently.
- **`js-modern-syntax` → `js-error-handling`** — error handling uses `cause` chaining (ES2022 from `js-modern-syntax`), async errors build on promise rejection model (async-js), and validation patterns feed directly into DOM event handlers (DOM fundamentals) and TS narrowing (TypeScript fundamentals). Placed here so DOM/TS code can assume error literacy.
- **`js-error-handling` → `js-iterators-generators`** — generators are a mechanism topic (suspend/resume, coroutines) that extends the async mental model. Needs iterator syntax from `js-modern-syntax`. Async generators + streams feed directly into Datastar's SSE consumption (Datastar fundamentals) and `notes/client-http/` streaming.
- **`js-iterators-generators` → DOM fundamentals** — DOM code uses classes, destructuring, optional chaining, iterators constantly; syntax fluency + iterator/generator mechanics mean DOM examples need no detours.
- **DOM fundamentals → TypeScript fundamentals** — TS leverages reading comprehension on every later course; types land concretely on the DOM APIs just learned.
- **TypeScript fundamentals → CSS layout fundamentals** — CSS is the biggest blind spot for backend-strong devs; closes the visible-UI gap.
- **CSS layout fundamentals → Browser rendering pipeline** — CRP needs CSSOM literacy to make sense; safely skippable since DOM capstone gives the lite version.
- **Browser rendering pipeline → Datastar fundamentals** — Datastar is the capstone; DOM events, SSE (from `notes/client-http/`), async generators (from `js-iterators-generators`), and CSS layout all feed directly into understanding hypermedia-driven reactivity without a virtual DOM layer in between.

## Off-ramps

- **Stop after CSS layout fundamentals** — ~4 months of work, reasonably literate, can review most UI work.
- **Stop after Datastar fundamentals (skipping Browser rendering pipeline)** — full path minus engine depth; hypermedia-driven UI covered.
- **Complete through Datastar fundamentals including Browser rendering pipeline** — full fundamentals depth; ready to architect, not just review.

## Rough timeline

At one chunk per session and a short session per day: ~6–7 months end-to-end after `js-inheritance` closes out. Off-ramping at TypeScript fundamentals cuts to ~4 months.

## Revising

- Update **Status** as courses progress / complete.
- Adjust scope chunk counts after each course's calibration.
- Move topics from **Off-path** into the main path if a project triggers them.
- Promote artifacts from `notes/` to `courses/` if a notes folder balloons past ~5 entries on the same theme.
- Bump the `Last updated` date when revising.
