# Error Handling & Defensive Patterns — Course TOC

Progress tracker for the course. Chunks grouped by theme.

## Calibration

_(To be filled at session start.)_

## Progress

- [ ] **Error fundamentals** — `Error` object anatomy (message, name, stack, cause), error types (`TypeError`, `RangeError`, `SyntaxError`, etc.), throw mechanics (any value, but why Error objects), stack trace construction
- [ ] **try/catch/finally** — control flow semantics, finally guarantees (runs even on return/throw), catch binding (optional catch), nested try blocks, performance considerations
- [ ] **Custom errors** — extending `Error`, preserving stack trace, `cause` chaining (ES2022), error hierarchies vs flat codes, `instanceof` checking and its prototype pitfalls
- [ ] **Quiz 1** (covering *Error fundamentals* through *Custom errors*): error anatomy, control flow, custom errors
- [ ] **Error propagation strategies** — throw-and-catch vs return-value (Result pattern), when to catch vs when to let bubble, error boundaries (logical groupings that catch and handle), re-throwing with context
- [ ] **Async error handling** — promise rejection vs throw, unhandled rejection events, `async`/`await` try/catch, error propagation through `.then` chains, `Promise.allSettled` for partial-failure tolerance
- [ ] **Quiz 2** (covering *Error propagation strategies* and *Async error handling*): propagation, async errors
- [ ] **Test 1** (part 1: covering *Error fundamentals* through *Async error handling*): error mechanics end-to-end
- [ ] **Validation & defensive input** — validate at boundaries (not everywhere), schema validation concept, fail-fast vs fail-safe, assertion functions, guard clauses, narrowing via validation (connects to TS narrowing in the *TypeScript fundamentals* course)
- [ ] **Graceful degradation** — fallback values, default behaviors, retry patterns, circuit breaker concept, user-facing vs developer-facing errors, error reporting (what to log, what to surface)
- [ ] **Patterns & anti-patterns** — silent swallows (empty catch), over-catching (catch-all at wrong level), error as control flow (when acceptable: Python EAFP vs JS), Pokemon exception handling, error codes vs error types
- [ ] **Quiz 3** (covering *Validation & defensive input* through *Patterns & anti-patterns*): validation, degradation, anti-patterns
- [ ] **Final test** (cumulative)
