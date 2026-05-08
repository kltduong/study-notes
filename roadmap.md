# Study Roadmap — Frontend Fundamentals

Goal: whole-system literacy (frontend included) for a backend-strong dev. Reviewing and architecting UI work alongside AI/devs, not specializing.

Last updated: 2026-05-07.

## Path

| #   | Course                     | Status        | Scope                                                                                                                                                      |
| --- | -------------------------- | ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `js-vars-scope`            | in progress   | Finish closures + variable lifecycle                                                                                                                       |
| 2   | `js-inheritance`           | in progress   | Finish chains, class syntax, composition (~30% remaining)                                                                                                  |
| 3   | DOM fundamentals           | new           | Tree, querying, mutation, event dispatch, delegation, attrs vs props, input, forms, custom events, observers, capstone with CRP-lite (~11 chunks)          |
| 4   | TypeScript fundamentals    | new           | Structural typing, primitives/literals, unions/intersections, generics + inference, narrowing, common utility types, working with libraries (~9–10 chunks) |
| 5   | CSS layout fundamentals    | new           | Box model, normal flow, positioning, stacking contexts, flexbox, grid, responsive, typography essentials (~8–10 chunks)                                    |
| 6   | Browser rendering pipeline | new, optional | Parse → CSSOM → render tree → layout → paint → composite, layer promotion, Core Web Vitals, profiling (~6–8 chunks)                                        |
| 7   | React fundamentals         | new           | Rendering model, components/props/state, derived state vs effects, lists/keys, forms, context, data fetching, composition (~10 chunks)                     |

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
- **Functions deep-dive (`this`/bind/composition)** — mostly covered by #1 and #2; remaining gaps are notes-level.

## Order rationale

- **1 → 2** — close open loops; preserve calibration data; promote `shaky` → `solid`.
- **2 → 3** — warm cache from `async-js`; substrate-before-abstractions matches formal-reasoning bias.
- **3 → 4** — TS leverages reading comprehension on every later course; types land concretely on the DOM APIs just learned.
- **4 → 5** — CSS is the biggest blind spot for backend-strong devs; closes the visible-UI gap.
- **5 → 6** — CRP needs CSSOM literacy to make sense; safely skippable since DOM capstone gives the lite version.
- **6 → 7** — React is the capstone; every prior pillar feeds into understanding why it works the way it does.

## Off-ramps

- **Stop after #5** — ~2 months of work, reasonably literate, can review most UI work.
- **Stop after #7 (skipping #6)** — full path minus engine depth.
- **Complete through #7 including #6** — full fundamentals depth; ready to architect, not just review.

## Rough timeline

At one chunk per session and a short session per day: ~3–4 months end-to-end after #2 closes out. Off-ramping at #5 cuts to ~2 months.

## Revising

- Update **Status** as courses progress / complete.
- Adjust scope chunk counts after each course's calibration.
- Move topics from **Off-path** into the main path if a project triggers them.
- Promote artifacts from `notes/` to `courses/` if a notes folder balloons past ~5 entries on the same theme.
- Bump the `Last updated` date when revising.
