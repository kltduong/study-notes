# Iterators, Generators & Async Iteration — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

_(To be filled at session start.)_

## Progress

- [ ] **Iterator protocol** — `Symbol.iterator`, iterator result object (`{ value, done }`), iterable vs iterator distinction, built-in iterables (Array, String, Map, Set), consuming iterables (`for...of`, spread, destructuring)
- [ ] **Writing custom iterables** — implementing `[Symbol.iterator]()`, stateful iterators, infinite sequences, iterator composition (zip, take, chain), early termination (`return()` method)
- [ ] **Quiz 1** (covering *Iterator protocol* and *Writing custom iterables*): iterator protocol, custom iterables
- [ ] **Generator fundamentals** — `function*` syntax, `yield` as suspend/resume, generator objects as iterators, lazy evaluation, memory efficiency vs eager collection
- [ ] **Generators as two-way channels** — `yield` as expression (receiving values via `.next(arg)`), `.throw()` and `.return()` for external control, coroutine pattern, generator-based state machines
- [ ] **`yield*` and composition** — delegating to sub-generators, flattening nested iteration, recursive generators (tree traversal), composing generator pipelines
- [ ] **Quiz 2** (covering *Generator fundamentals* through *`yield*` and composition*): generator mechanics, two-way communication, delegation
- [ ] **Test 1** (part 1: covering *Iterator protocol* through *`yield*` and composition*): iterators and generators end-to-end
- [ ] **Async iterators & `for await...of`** — `Symbol.asyncIterator`, async iterator protocol, `for await...of` semantics, consuming async data sources (paginated APIs, event streams)
- [ ] **Async generators** — `async function*`, `yield` with promises, real-world patterns (polling, SSE consumption, chunked processing), backpressure concept (pull-based vs push-based)
- [ ] **Quiz 3** (covering *Async iterators & `for await...of`* and *Async generators*): async iteration, async generators
- [ ] **Streams & integration** — `ReadableStream` as async iterable, transform streams, connecting generators to DOM (progressive rendering), generator-to-observable bridge concept, when to use generators vs streams vs callbacks
- [ ] **Final test** (cumulative)
