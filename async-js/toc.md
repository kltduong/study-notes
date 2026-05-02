# Asynchronous JavaScript: Promises, Callbacks, Async Await

## Calibration

**Starting point:** Knows callbacks and promises exist, has used async/await. Fuzzy on how they actually work under the hood — event loop, microtasks, task queues, execution ordering.

**Goal:** Clean mental model of how async JS actually works — event loop, task queues, why things run in the order they do.

**Planned arc:**

1. Foundations (chunks 1–2) — why async exists, event loop model (call stack + task queue). Moderate pace; this is where the fuzziness lives.
2. Callbacks (chunks 3–5) — original async pattern, error handling, callback hell. Lighter pace; focus on _why_ and pain points.
3. Promises (chunks 6–11) — states, chaining, combinators, testing. Moderate-to-slow; build the scheduling mental model.
4. Async/Await & Advanced Event Loop (chunks 12–15) — syntax sugar, microtasks vs macrotasks, task priorities, rAF. Slower pace; final layer of the execution-order model.

Throughout: emphasis on predicting execution order as the proof the mental model works.

## Course Progress

### Part 1 — Foundations & Callbacks

- [x] **Chunk 1: Intro & What Is Asynchronicity?**
      Introduction; Structure of This Course; What Is Asynchronicity?; Typical Example of an Asynchronous Action in JavaScript; Synchronous vs Asynchronous in JavaScript; Quick Note about Github Repository.
      📊 solid
      🔧 Refinements: - Said "the code is routed" when it's the _callback function_ that gets placed in the task queue, not the code inside it

- [x] **Chunk 2: Event Loop Basics**
      Event Loop in JavaScript — Call Stack and Task Queue; Let's Fix Our Example.
      📊 solid
      🔧 Refinements: - Said "engine places callback into the task queue" — it's the runtime that enqueues, not the engine - Said two same-delay setTimeout callbacks aren't guaranteed ordered — they are, because the task queue is FIFO and the runtime enqueues in registration order

- [x] **Chunk 3: Callbacks**
      What Is a Callback In JavaScript?; Callbacks Are Not Always Asynchronous; How To Handle Errors In Asynchronous Code; Pros & Cons Of Callbacks; Callback Examples In JavaScript Libraries.
      📊 solid

  → [x] Quiz (chunks 2–3): event loop mechanics, callback fundamentals, error handling patterns.
  📊 solid

- [x] **Chunk 4: Callback Problems & HTTP**
      Callbacks Lack Readability; Making HTTP Requests From Your Browser; Callback Hell.
      📊 solid

- [x] **Chunk 5: Testing Callbacks**
      Setting Up Testing Environment; Testing Callback Functions With Mocha And Chai.
      📊 shaky — got the timeout consequence right but initially thought missing `done` causes a timeout (it actually causes a silent false pass); arrow function `this` reasoning was close but imprecise on the mechanism (said "global object" rather than lexical capture from enclosing scope)
      🔧 Refinements: - Said missing `done` parameter causes a timeout — Mocha actually treats it as sync and passes immediately (false positive) - Said arrow function `this` falls back to global object — it captures `this` from the enclosing lexical scope at definition time, which may or may not be global
      📊 shaky → solid — nailed both weak spots on review sweep

  → [x] Test — Part 1 (chunks 1–5): async fundamentals, event loop, callbacks end-to-end.
  📊 solid — missed error scattering as a distinct structural problem (lumped it with composition/readability)
  🔧 Refinements: - Listed nesting and readability as two separate problems instead of identifying the three distinct structural issues: inversion of control, inside-out composition, and scattered error handling

### Part 2 — Promises

- [x] **Chunk 6: Promise Fundamentals**
      What Is a Promise In JavaScript; How To Create A Promise; Final States Of The Promise; How To Consume JavaScript Promises: Promise.then.
      📊 shaky → solid — initially said missing onRejected handler swallows the rejection (it propagates); nailed the distinction on follow-up
      🔧 Refinements: - Said missing onRejected in .then() silently swallows the rejection — it actually propagates through the chain until handled or surfaces as an unhandled rejection

- [x] **Chunk 7: Working with Promises**
      Rewriting Our Function Using Promises; How To Promisify any JavaScript Function; Chaining Promises.
      📊 solid — hardcoded values in promisification wrapper instead of passing through originals; corrected quickly
      🔧 Refinements: - Hardcoded `reject(new Error("No city"))` and `resolve(22)` instead of passing through `err` and `value` — wrapper should be general-purpose, not tied to one call

  → [ ] Quiz (chunks 6–7): promise states, .then, chaining, promisification.

- [ ] **Chunk 8: Fetch & Rejection Handling**
      Making HTTP Requests Using Fetch API; How To Avoid Callback Hell; Handling Promise Rejections; Promise.resolve and Promise.reject.

- [ ] **Chunk 9: Promise.all & Promise.allSettled**
      Executing Promises In Parallel: Promise.all; How Promise.all Handles Rejections; Promise.all: Implementing From Scratch; Executing Promises In Parallel: Promise.allSettled; Promise.allSettled: Implementing From Scratch.

  → [ ] Quiz (chunks 8–9): fetch, rejection flows, parallel combinators.

- [ ] **Chunk 10: Promise.race & Promise.any**
      Which Promise Is Faster? Promise.race; Getting First Successful Promise: Promise.any; Promise.race: Implementing From Scratch; Promise.any: Implementing From Scratch.

- [ ] **Chunk 11: Testing Promises**
      Setting Up Testing Environment; Testing JavaScript Promises Using Mocha And Chai; Timeouts In Mocha; Making Multiple Promise Assertions In One Test.

  → [ ] Test — Part 2 (chunks 6–11): promises end-to-end, combinators, testing.

### Part 3 — Async/Await & Advanced Event Loop

- [ ] **Chunk 12: Async/Await**
      Async Functions in JavaScript; Await Keyword in JavaScript; Using Async Await with Fetch API; Top Level Await; Handling Errors Using Async Await; Sequential vs Parallel Execution.

  → [ ] Quiz (chunk 12): async/await mechanics, error handling, sequential vs parallel.

- [ ] **Chunk 13: Task Priorities & Microtasks**
      What About Task Priorities?; A Closer Look at the Task Queue; Handling Long Tasks in JavaScript Part 1; Handling Long Tasks in JavaScript Part 2; Microtasks.

- [ ] **Chunk 14: Debugging & Animations**
      Debugging Our Initial Example; Animations; Creating Animations Using requestAnimationFrame.

  → [ ] Quiz (chunks 13–14): microtasks vs macrotasks, task scheduling, rAF.

- [ ] **Chunk 15: Wrap-Up**
      Quick Recap of Event Loop in the Browser; Summary.

  → [ ] Test — Part 3 (chunks 12–15): async/await, advanced event loop, microtasks, animations.

  → [ ] **Final Test** (cumulative, full course).
