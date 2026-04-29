# Asynchronous JavaScript

How async actually works in JS — not just the syntax, but the execution model: event loop, task queues, and why things run in the order they do. The unifying thread is that JS engines are single-threaded, and everything async is built on top of that constraint via the runtime and event loop.

## Reading Order

Start with [what-is-async](what-is-async.md) for the foundational "why" — why async exists, the three-part model (engine / runtime / event loop). Then [event-loop-basics](event-loop-basics.md) zooms into the mechanics of the call stack and task queue. [callbacks](callbacks.md) and [callback-problems](callback-problems.md) cover the original async pattern and why it needed replacing.

## Notes

- [what-is-async.md](what-is-async.md) — Why async exists: single-threaded engines, the runtime delegation model, sync vs async.
- [event-loop-basics.md](event-loop-basics.md) — Call stack, task queue, event loop algorithm, execution tracing, ordering guarantees.
- [callbacks.md](callbacks.md) — What callbacks are, sync vs async callbacks, error-first convention, pros/cons.
- [callback-problems.md](callback-problems.md) — Why callbacks break down: composition pain, callback hell, XHR, and the limits that motivated promises.

## Cross-Cutting Concepts

- **Single-threaded engine, multi-threaded runtime** — the engine only does one thing at a time; the runtime handles I/O on separate threads. This distinction drives everything.
- **"Later" not "parallel"** — async means deferred execution, not concurrent execution within the engine.
- **The stack must be empty** — the event loop's one rule. No callback runs until the call stack drains. This explains every "surprising" execution order.
- **try/catch can't cross async boundaries** — by the time an async callback runs, the original call stack (and any `try` block on it) is gone. This is why error-first conventions and later promises exist.
