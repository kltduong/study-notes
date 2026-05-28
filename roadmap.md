# Study Roadmaps

Three tracks toward full-stack architectural literacy.

Last updated: 2026-05-28.

## Active

- **Frontend Fundamentals** → `js-hof-functional` (27%)
- **Dataflows & Reactive** → not started (blocked: needs `js-hof-functional` + `js-iterators-generators`)
- **Systems & Infrastructure** → not started

---

## Frontend Fundamentals

Goal: whole-system UI literacy for a backend-strong dev. Reviewing and architecting UI work alongside AI/devs, not specializing.

| #   | Course                     | Status        | Progress | Scope                                                                                                                                                      |
| --- | -------------------------- | ------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `js-vars-scope`            | in progress   | 92%      | Closures + variable lifecycle                                                                                                                              |
| 2   | `js-inheritance`           | in progress   | 73%      | Chains, class syntax, composition                                                                                                                          |
| 3   | `js-values-fn-this`        | in progress   | 73%      | Values & memory model, Reference type, `this` determination, `call`/`apply`/`bind`, arrow functions, `new` protocol (~8 chunks)                            |
| 4   | `js-hof-functional`        | in progress   | 27%      | Functions as values, map/filter/reduce, composition & pipelines, currying, algebraic structure, FP vs OOP (~9 chunks)                                       |
| 5   | `js-modules`               | in progress   | 0%       | ES module syntax, static linking, live bindings, module lifecycle, circular deps, dynamic `import()`, bundlers (~13 chunks)                                 |
| 6   | `js-modern-syntax`         | new           | 0%       | Class fields, private `#`, destructuring, spread/rest, `?.`, `??`, iterators, generators, symbols, proxy/reflect (~12 chunks)                              |
| 7   | `js-error-handling`        | new           | 0%       | Error anatomy, try/catch/finally, custom errors, propagation, async errors, validation (~8 chunks)                                                         |
| 8   | `js-iterators-generators`  | new           | 0%       | Iterator protocol, generators, two-way channels, `yield*`, async iterators, async generators, streams (~8 chunks)                                          |
| 9   | DOM fundamentals           | new           | 0%       | Tree, querying, mutation, events, delegation, forms, observers, capstone (~11 chunks)                                                                      |
| 10  | TypeScript fundamentals    | new           | 0%       | Structural typing, unions/intersections, generics, narrowing, utility types (~9–10 chunks)                                                                 |
| 11  | CSS layout fundamentals    | new           | 0%       | Box model, flow, positioning, flexbox, grid, responsive (~8–10 chunks)                                                                                     |
| 12  | Browser rendering pipeline | new, optional | 0%       | Parse → CSSOM → render tree → layout → paint → composite, Core Web Vitals (~6–8 chunks)                                                                    |
| 13  | Datastar fundamentals      | new           | 0%       | Hypermedia-driven reactivity, signals, SSE, morphing, forms & CRUD (~8–10 chunks)                                                                          |

---

## Dataflows & Reactive Systems

Goal: build the mental model that iteration, streaming, reactivity, and distributed processing are the same structural pattern at increasing scale.

Prerequisites: `js-hof-functional` + `js-iterators-generators` (from Frontend path) before course 1.

| #   | Course                    | Status | Progress | Scope                                                                                                                     |
| --- | ------------------------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | `dataflow-foundations`    | new    | 0%       | Transducers, reducer algebra, push vs pull, iterator combinators. JS + Python + Haskell notation (~6–8 chunks)            |
| 2   | `async-streams`           | new    | 0%       | Async iterators, backpressure, observable pattern (RxJS), hot vs cold, cancellation (~6–8 chunks)                         |
| 3   | `reactive-systems`        | new    | 0%       | Signals, fine-grained dependency tracking, glitch-free propagation, SolidJS/Vue internals (~5–6 chunks)                   |
| 4   | `distributed-streams`     | new    | 0%       | Event logs (Kafka), partitioning, windowing, exactly-once, stateful processing (Flink/Beam), replay (~6–8 chunks)         |
| 5   | `workflow-orchestration`  | new    | 0%       | DAGs, durable execution (Temporal), saga patterns, Step Functions, idempotency (~5–6 chunks)                              |
| 6   | `dataflow-theory`         | new    | 0%       | Capstone — monoids/folds/functors as unifying algebra, incremental computation, stream algebra (~4–5 chunks)              |

---

## Systems & Infrastructure

Goal: design and operate production systems. Understand how things run, fail, and scale.

No hard prerequisites from other tracks. Can start independently or project-triggered.

| #   | Course                     | Status | Progress | Scope                                                                                                                     |
| --- | -------------------------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------- |
| 1   | `networking-fundamentals`  | new    | 0%       | TCP/IP, DNS, HTTP/1.1→2→3, TLS, connection pooling, socket lifecycle (~7–8 chunks)                                        |
| 2   | `linux-internals`          | new    | 0%       | Process model, virtual memory, I/O models (epoll, io_uring), syscalls, signals (~7–8 chunks)                              |
| 3   | `api-design`               | new    | 0%       | REST constraints, resource modeling, hypermedia, versioning, GraphQL tradeoffs (~5–6 chunks)                               |
| 4   | `auth-architecture`        | new    | 0%       | OAuth2/OIDC, JWT mechanics, session models, RBAC/ABAC, multi-tenant auth (~5–6 chunks)                                    |
| 5   | `database-internals`       | new    | 0%       | Storage engines, indexing, query planning, isolation levels, MVCC, replication (~6–8 chunks)                               |
| 6   | `containers-orchestration` | new    | 0%       | Container internals (namespaces, cgroups), Docker, K8s mental model, service mesh (~6–7 chunks)                           |
| 7   | `distributed-systems`      | new    | 0%       | CAP/PACELC, consistency models, consensus (Raft), failure detection, sagas, CRDTs (~7–8 chunks)                           |
| 8   | `observability`            | new    | 0%       | Traces, metrics, logs, OpenTelemetry, alerting, SLO/SLI/error budgets (~5–6 chunks)                                       |
| 9   | `cloud-patterns`           | new    | 0%       | Event-driven architecture, CQRS, cell-based design, serverless, cost modeling (~6–7 chunks)                               |

---

## Revising

- Update **Status** and **Progress** as courses progress / complete.
- Adjust scope chunk counts after each course's calibration.
- Update the **Active** section when switching focus or starting a new track.
