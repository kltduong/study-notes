# 1. Patterns & Pitfalls — teaching draft

## 1.1. Plan (teaching order)

- [x] **0. Teaser** — DOM `addEventListener` extraction; surprising `this` (not `undefined`)
- [x] **1. The pitfall sites** — catalog of where extraction happens (callbacks, event handlers, setTimeout, destructuring, higher-order)
- [x] **2. The DOM exception** — Web IDL "invoke a callback" sets `this = currentTarget`; consequences
- [x] **3. The fix spectrum** — arrow at call site, arrow field, `bind`-in-constructor, `EventListener` object protocol; status/when to use
- [ ] **4. Decision matrix** — pulling pitfall sites + DOM exception + fix spectrum into a "which fix for which site" table
- [ ] **5. Worked synthesis** — one example showing 3+ sites + 3+ fixes side-by-side

## 1.2. Teaser

```js
"use strict";                                                          // L1
class Logger {                                                         // L2
  prefix = "[LOG]";                                                    // L3
  log() { console.log(`${this.prefix} event fired`); }                 // L4
}                                                                      // L5

const l = new Logger();                                                // L6
const button = new EventTarget();                                      // L7 — pretend this is a DOM button
button.addEventListener("click", l.log);                               // L8
button.dispatchEvent(new Event("click"));                              // L9 — what prints?
```

Predict the output of L9 and *why* — be precise about the mechanism.

Three plausible answers — pick one and explain:

1. **TypeError** — `Cannot read properties of undefined (reading 'prefix')`
2. **`"[LOG] event fired"`** — works correctly
3. **`"undefined event fired"`** — runs but prints `undefined`

Which mechanism produces your prediction? (Reference base on `addEventListener`'s side, the dispatch algorithm, the runtime, …?)


### 1.2.1. Reveal

Actual output of L9:

```
undefined event fired
```

Not a TypeError. The string `"undefined"` came from interpolating `${undefined}` into the template literal — `this.prefix` evaluated to plain `undefined` and read fine, because `this` itself was *not* `undefined`.

The standard extraction-loses-`this` reasoning predicts:

```
this = undefined → this.prefix → TypeError
```

That reasoning is correct **for a plain runtime call** — the kind a generic "schedule this function" mechanism would do. The piece it's missing is that `dispatchEvent` is not a plain call. It's a DOM-spec algorithm that supplies `thisValue` explicitly.

#### 1.2.1.1. What dispatch actually does

The DOM standard's event dispatch algorithm calls the listener through Web IDL's **"invoke a callback function"** operation. The relevant step (paraphrased):

> Call the callback's `[[Call]]` internal method with **`thisValue = the EventTarget being dispatched on`** and the event arguments.

Effectively, `target.dispatchEvent(evt)` calls `listener.[[Call]](target, [evt])` — equivalent to `listener.call(target, evt)`, not `listener(evt)`. The `addEventListener` API specifies `currentTarget` as the `this` value for the callback.

So in our snippet:

```js
button.addEventListener("click", l.log);   // L8 — extraction (Reference discarded), bare function stored
button.dispatchEvent(new Event("click"));  // L9 — calls l.log.[[Call]](button, [event])
```

Inside `log`, `this = button` (an `EventTarget`). `button.prefix` doesn't exist → reads as `undefined` — no error, just a missing property. Template literal interpolates `undefined` to the string `"undefined"`.

#### 1.2.1.2. Why this matters

The extraction *did* happen. The Reference was lost. But the next call site **isn't a plain call** — it's a DOM-driven call with a built-in `this` override. So the failure mode is different:

| Caller of extracted function | What it does | `this` inside method | Symptom |
|---|---|---|---|
| Plain call (`fn()`, `setTimeout(fn, 0)`, `forEach(fn)`) | Identifier-based `[[Call]]`, no override | `undefined` (strict) | TypeError on first property read |
| DOM event dispatch | Web IDL invoke with `thisValue = currentTarget` | The EventTarget | **Silent** wrong-object behavior — undefined property reads, no errors |
| Native `bind`-style dispatchers (rare) | Varies by API contract | API-specific | Varies |

DOM events are the dominant case where extraction produces a *silent* bug instead of a loud TypeError. That's what makes them especially nasty:

- Loud failure (`fn()` → TypeError) tells you *immediately* that something's wrong.
- Silent failure (`addEventListener` → `target.prefix → undefined`) means the handler "runs" — and your debugging starts with "why isn't the counter incrementing?" rather than "where's the TypeError?".

#### 1.2.1.3. Tying it back to the pipeline

This is consistent with everything we've built. `dispatchEvent` is just another caller; the only question is "what does the caller pass to `[[Call]]` as `thisValue`?":

- Reference-base call site → uses `[[Base]]`
- Plain identifier call site → ER → strict-`undefined` (or sloppy-`globalThis`)
- `func.call(arg)` / `func.apply(arg, ...)` → uses `arg`
- BoundFunction wrapper → uses `[[BoundThis]]`, ignores incoming
- **DOM event dispatch → uses `currentTarget`** (specified by the DOM standard, not ECMAScript)

The mechanism itself is unchanged — only the source of `thisValue` varies. Adding "DOM dispatch" to the list of `thisValue` suppliers is the small addition. The big lesson is the *failure-mode shift*: silent vs loud.



## 1.3. The pitfall sites

The *Teaser* showed one site (DOM dispatch). The full catalog: every call site that takes a function value, stores it, and invokes it later through *some* dispatcher. The dispatcher decides what `this` is — your code has no say once the function is handed over.

Three things have to happen for a `this`-loss bug:

1. **Extraction** — a member expression is evaluated in a position that requires `GetValue` (argument, assignment, destructuring, return). The Reference is discarded; only the bare function object survives.
2. **Storage** — the function is held somewhere (parameter slot, array, listener registry, timer queue, microtask queue).
3. **Re-invocation** — a dispatcher later calls the function via *its own* `thisValue` choice — not the original object.

Steps 1 and 3 can be far apart in the source — that's what makes these bugs slippery.

### 1.3.1. Extraction shapes

All of these run `GetValue` on a member expression and discard the Reference. After this point, the function is a bare object — no memory of where it came from.

| Shape | Example | Why it's extraction |
|---|---|---|
| **Variable assignment** | `const fn = obj.method;` | RHS evaluated → GetValue |
| **Argument passing** | `register(obj.method)` | Each argument expression GetValue'd |
| **Array/object literal** | `const handlers = [obj.m1, obj.m2];` | Each element GetValue'd |
| **Destructuring** | `const { method } = obj;` | Desugars to `const method = obj.method` |
| **Return value** | `function get() { return obj.method; }` | Return value GetValue'd |
| **Spread into args** | `register(...[obj.method])` | Array elements GetValue'd, then spread |
| **Default parameter** | `function f(fn = obj.method) {}` | Default expression GetValue'd if param omitted |

What is **not** extraction (Reference preserved):

| Non-extraction | Why |
|---|---|
| `obj.method()` | The `()` operator is the consumer; Reference goes straight to it |
| `obj.method?.()` | Optional-call preserves Reference (covered earlier in [`this` determination](this-determ.md)) |
| `(obj.method)()` | Parentheses around a Reference don't trigger GetValue |
| `obj.method.bind(obj)` | `.bind` is invoked *as a method on the function*, then the BoundFunction stores `obj` |

### 1.3.2. Re-invocation sites — what each dispatcher passes as `this`

Once a bare function is stored, its eventual call site decides `thisValue`. The interesting part: **most real dispatchers aren't plain calls.** Each environment-supplied dispatcher has its own contract.

| Dispatcher | `thisValue` it passes | Failure mode on extraction |
|---|---|---|
| Plain identifier call (`fn()`) | ER base → `undefined` (strict) / `globalThis` (sloppy) | Loud TypeError on first property read (strict) |
| `Array.prototype.forEach(fn)` | `undefined` (strict) — unless 2nd `thisArg` passed | Loud TypeError |
| `Array.prototype.map(fn)` / `.filter(fn)` / `.find(fn)` etc. | Same — `undefined` unless `thisArg` passed | Loud TypeError |
| `setTimeout(fn, ms)` / `setInterval(fn, ms)` | **Host-dependent.** Browsers: `globalThis` (Window). Node: a `Timeout` object | Silent wrong-object — no errors, just nonsense |
| `Promise.then(fn)` / `.catch(fn)` | `undefined` (per spec) | Loud TypeError |
| `queueMicrotask(fn)` | `undefined` | Loud TypeError |
| `requestAnimationFrame(fn)` | `globalThis` (browser) | Silent wrong-object |
| `EventTarget.dispatchEvent` (via `addEventListener`) | `currentTarget` | **Silent wrong-object** (the *Teaser reveal* above) |
| `node.on("event", fn)` (Node EventEmitter) | The `EventEmitter` instance | Silent wrong-object |
| Web Worker `postMessage` handler | The `Worker`/global scope | Silent wrong-object |
| `MutationObserver` callback | The observer instance | Silent wrong-object |
| Constructor (`new fn(...)`) | A fresh object (then return-value override applies) | Probably TypeError, possibly silent if body tolerates fresh `{}` |
| Generator `next(arg)` | The generator object | Silent wrong-object |

The pattern: **runtime-internal dispatchers (Promise, microtask, array methods)** stay close to the spec's "plain call" default and pass `undefined`. **Host-environment dispatchers (DOM, Node EventEmitter, timers)** typically supply something — usually whatever the environment thinks is the "natural target." That something is almost never the object the method came from.

### 1.3.3. The two failure shapes

This is the practical takeaway from the catalog:

| Shape | When | Symptom | Debug experience |
|---|---|---|---|
| **Loud failure** | Plain calls, array methods, Promise/microtask | TypeError on first `this.X` read | Stack trace points at the method body — relatively easy |
| **Silent failure** | DOM events, EventEmitter, host timers, observers | `this.X` reads `undefined`; no error; downstream logic gets garbage | Bug surfaces far from the cause — counter doesn't tick, state doesn't update, render is wrong |

Strict mode (mandatory in classes/modules) made the **loud** path louder — a property access on `undefined` throws immediately rather than reading from `globalThis`. But strict mode does nothing for the **silent** path: the dispatcher already supplied a `this`, so `this` is a real object — properties just don't exist on it.

So: arrows / `bind` / etc. exist not just to "preserve `this` for safety" — they exist to flip silent bugs into correct behavior, and (where a fix is missing) to flip silent bugs into loud ones.

> **Aside —** Several entries in *Re-invocation sites* are themselves subject to spec evolution. Pre-ES2015, `Array.prototype.forEach`'s `this` for the callback was sloppy-`globalThis`; under strict it's `undefined`. The DOM's "use `currentTarget`" rule has been there since DOM Level 2 and isn't going to change, but the rule for *new* host APIs ("what `this` should we pass?") is still made case-by-case by spec authors. The mental model — "find the dispatcher, find its contract" — generalizes; the table doesn't.


## 1.4. The DOM exception — why dispatch passes `currentTarget`

The *Teaser reveal* stated the rule: DOM event dispatch calls listeners with `this = currentTarget`. This section unpacks *why* — both the spec mechanism and the historical motivation. Once both layers are clear, the silent-failure pattern stops looking arbitrary.

### 1.4.1. The minimum event model (just-enough scope)

DOM events get a full course later (the *DOM fundamentals* course on the roadmap — capture/bubble phases, delegation, custom events, observers). For this chunk, only three things matter:

- **`EventTarget`** — an object that can have listeners registered on it. In the browser, every DOM element is an `EventTarget`. In Node 19+, `EventTarget` is a built-in global class used for examples here (so the snippets run without a browser).
- **`addEventListener(type, listener)`** — registers `listener` for events of name `type` (e.g. `"click"`). Registering the same listener twice for the same type is a silent no-op.
- **`dispatchEvent(event)`** — synchronously calls every registered listener whose type matches `event.type`. Each call is roughly `listener.[[Call]](currentTarget, [event])` — i.e. equivalent to `listener.call(currentTarget, event)`. `currentTarget` is the `EventTarget` `addEventListener` was called on.

That is the entire dispatcher we need. No capture/bubble phases, no propagation, no `event.stopPropagation()`. We use `dispatchEvent` only because it's the simplest concrete dispatcher whose `thisValue` contract isn't `undefined` — it lets us study `this` extraction onto a host-supplied dispatcher without setting up a browser.

```js
"use strict";                                                          // L1
const target = new EventTarget();                                      // L2 — node we attach listeners to
target.addEventListener("ping", function (event) {                     // L3 — register
  console.log("got", event.type, "this is target?", this === target);  // L4
});                                                                    // L5
target.dispatchEvent(new Event("ping"));                               // L6 → "got ping this is target? true"
```

L6 trace: dispatch finds the one registered listener, calls it as `listener.[[Call]](target, [event])`. Inside the listener, `this = target`, `event` is the regular argument.

> 🔖 Later: capture vs bubble phases, `event.target` vs `event.currentTarget` (they differ on bubbled events), `stopPropagation`, custom events, observer APIs — all in the *DOM fundamentals* course. The single-target model above is sufficient for studying `this` determination here.

### 1.4.2. Two specs, two layers (where the contract comes from)

JS doesn't define `addEventListener` — the **DOM standard** does (a separate W3C/WHATWG spec layered on top of ECMAScript). The DOM standard composes ECMAScript via **Web IDL** — the interface description language used to translate spec-level operations into JS-callable methods.

| Spec | What it owns | Where it lives |
|---|---|---|
| **ECMAScript** | `[[Call]]`, `[[Construct]]`, the Reference type, OrdinaryCallBindThis, strict mode | tc39.es/ecma262 |
| **Web IDL** | "invoke a callback function" — the bridge that calls JS functions from spec algorithms | webidl.spec.whatwg.org |
| **DOM** | `EventTarget`, `addEventListener`, dispatch algorithm | dom.spec.whatwg.org |

The DOM standard says (paraphrased): "during dispatch, for each registered listener, **invoke the callback** with the event as the only argument and **`this` set to `currentTarget`**." The phrase "invoke the callback" is a hyperlink into Web IDL, which in turn invokes ECMAScript's `[[Call]]` with the supplied `thisValue`.

So at the JS level, what runs is essentially:

```js
// Conceptual — not actual API:
listener.[[Call]](currentTarget, [event])
```

This is structurally identical to `listener.call(currentTarget, event)` from your perspective. The DOM spec just chose `currentTarget` as the value; the rest of the pipeline is unchanged.

### 1.4.3. Why `currentTarget`? (motivation)

The choice predates classes, modules, and arrow functions. In the late-1990s DOM Level 0 / Level 2 era, the dominant pattern was:

```js
button.onclick = function (event) {                                    // L1
  this.disabled = true;                                                // L2 — `this` = the button element; sets HTMLButtonElement.disabled = true
};                                                                     // L3 — `event` is the separate first argument
```

Two slots, two mechanisms:

- **`this`** — supplied by the DOM dispatcher; here it's `currentTarget`, i.e. `button` (the element listener was attached to).
- **`event`** — passed as a regular argument (parameter index 0); the `Event` object describing what happened.

`this.disabled = true` writes to `button.disabled` — a property on `HTMLButtonElement` that greys the button out and stops further clicks. Nothing to do with disabling the event itself (event cancellation is `event.preventDefault()` / `event.stopPropagation()`).

A common idiom built on this: "click once and disable" submit buttons, where the inline handler mutates the very element it's attached to.

> 🔖 Later: DOM event mechanics in depth (capture/bubble phases, `currentTarget` vs `target`, delegation, custom events, observers) live in course (*DOM fundamentals*). Here we only need the `this` contract — the dispatcher passes `currentTarget`, full stop.

In **modern** code with classes, that convention reverses: `this` "should" be the controller object that owns state, not the DOM node. So the DOM's well-meaning convenience has become the most common silent-`this`-loss source in app code.

The behavior is correct per spec; what changed is what programmers want `this` to mean. **Status:** legacy convention. Don't write new code that relies on `this = currentTarget` inside a listener — use `event.currentTarget` explicitly when you need the element, and `this` for your controller object (with one of the fixes from *the fix spectrum* below).

### 1.4.4. Other host dispatchers follow the same template

The DOM is the most common case but not the only one. Each host environment defines its own `thisValue` contract for the callbacks it dispatches. The mental model: **find the dispatcher's spec, look up the contract.**

| API | Where the contract lives | `thisValue` |
|---|---|---|
| `EventTarget.dispatchEvent` | DOM standard | `currentTarget` |
| Node `EventEmitter` | Node docs (events module) | The emitter instance |
| `MutationObserver` | DOM standard | The observer instance |
| `IntersectionObserver` | Intersection Observer spec | The observer instance |
| `setTimeout` callback | HTML spec (browsers) / Node docs | `globalThis` (browser) / `Timeout` object (Node) |
| `requestAnimationFrame` | HTML spec | `globalThis` |
| `Promise.then` callback | ECMAScript | `undefined` (per spec — no host override) |

Pure-JS dispatchers (Promise, microtask, Array methods) stay close to the ECMAScript default of `undefined`. **Host-driven dispatchers tend to supply something** — almost never the object you wrote `obj.method` for. Treat any host API that takes a function as a candidate for silent `this`-loss until the spec confirms otherwise.


## 1.5. The fix spectrum

The previous sections (*pitfall sites* and *the DOM exception*) mapped *where* `this` goes wrong. This section maps *how* to make it right. Five mechanisms — five different tradeoffs. None is universal best; the right pick depends on the call shape, memory profile, and whether you'll need to remove the listener.

A fix works by inserting one of:

- **A wrapper that locks `this`** before the dispatcher gets the function (`bind`, arrow at call site, arrow field).
- **No wrapper at all** — keep the call shape as a member expression so the standard Reference-base rule does the work (`EventListener` object protocol, inline arrow that *does the call itself*).

Both shapes solve the same problem (preventing extraction-then-rebind). They differ in object identity, memory cost, and whether `this` can be overridden later.

### 1.5.1. Arrow at call site — wrap the extraction inline

The minimal fix. Replace the extraction with an inline arrow that performs the call:

```js
"use strict";                                                          // L1
class Logger {                                                         // L2
  prefix = "[LOG]";                                                    // L3
  log(msg) { console.log(`${this.prefix} ${msg}`); }                   // L4
}                                                                      // L5

const l = new Logger();                                                // L6
const button = new EventTarget();                                      // L7
button.addEventListener("click", (event) => l.log(event.type));        // L8
button.dispatchEvent(new Event("click"));                              // L9 → "[LOG] click"
```

Mechanism trace at L8:
- The arrow is the listener. The DOM dispatcher calls the *arrow*, not `l.log`. The arrow's `this` is irrelevant (it's lexical) — what matters is the body.
- Inside the body: `l.log(event.type)` is a **member-expression call** — Reference base = `l` → `this = l` inside `log`. Standard Reference-base rule.
- The dispatcher's choice of `thisValue` is wasted on the arrow (arrows ignore it), but the arrow then makes the *correct* call.

| Aspect | Verdict |
|---|---|
| **Memory cost** | One arrow per registration site (typically not per instance — depends on enclosing scope) |
| **`this`-locking strength** | Strong — arrow is structurally immune (covered in [arrow-fns](arrow-fns.md)) |
| **Removable?** | Only if you keep the arrow reference in a variable: `const handler = (e) => l.log(e.type); button.addEventListener("click", handler); button.removeEventListener("click", handler);` |
| **Status** | Default for one-off listener registration. Most readable for "convert this dispatcher's call into a member-expression call." |

Trade-off: the arrow body has to translate the dispatcher's argument shape into your method's argument shape. For a parameterless method this is just `() => l.log()`; for one that needs the event, `(e) => l.log(e.type)`. Sometimes verbose, but always explicit.

### 1.5.2. Arrow field — lock `this` per instance, structurally

Already introduced in [Class & `this`](class-deep-dive.md), in the *arrow-field pattern* section. Recap in this context: define the method as a class field with an arrow function. The arrow captures `this = the instance` at construction time and is structurally immune to extraction-rebinding.

```js
"use strict";                                                          // L1
class Logger {                                                         // L2
  prefix = "[LOG]";                                                    // L3
  log = (msg) => console.log(`${this.prefix} ${msg}`);                 // L4 — arrow field
}                                                                      // L5

const l = new Logger();                                                // L6
const button = new EventTarget();                                      // L7
button.addEventListener("click", l.log);                               // L8 — extraction is fine!
button.dispatchEvent(new Event("click"));                              // L9 — but arg shape mismatch
```

Wait — what does L9 print?

`l.log` is the arrow field, which expects `(msg)`. The dispatcher calls it with `(event)` as the argument. So `msg = event` (an `Event` object), and template-literal stringification of an `Event` typically gives `[object Event]`. The output is `"[LOG] [object Event]"` — not exactly what was wanted, but no `this`-loss, no TypeError.

The arrow-field fix solves *`this`*-loss but doesn't change the dispatcher's argument-passing. Your method has to either accept the dispatcher's argument shape directly, or you wrap (back to the *arrow at call site* fix above).

| Aspect | Verdict |
|---|---|
| **Memory cost** | One function object per instance (vs one per class for prototype methods) |
| **`this`-locking strength** | Strong — structural; survives any number of extractions, `call`/`apply`/`bind` |
| **Removable?** | Yes — `l.log` is the same identity every time it's read (it's a property on the instance) |
| **`super`-callable?** | No (arrows have no `[[HomeObject]]`) |
| **Subclass override?** | Awkward — must be a field, not a method, in the subclass too (the *what goes wrong if you reach for arrow fields by default* section in [Class & `this`](class-deep-dive.md)) |
| **Status** | Default when the method *will* be passed as a callback. If you control the class and the method's main purpose is being handed to dispatchers, arrow field is the right shape. |

### 1.5.3. `bind`-in-constructor — pre-class workaround, still legal

The historical alternative to arrow fields. Inside the constructor, replace each method with a bound version:

```js
"use strict";                                                          // L1
class Logger {                                                         // L2
  prefix = "[LOG]";                                                    // L3
  constructor() {                                                      // L4
    this.log = this.log.bind(this);                                    // L5 — bind in constructor
  }                                                                    // L6
  log(msg) { console.log(`${this.prefix} ${msg}`); }                   // L7
}                                                                      // L8
```

Mechanism: `this.log` on L5 RHS resolves to the prototype method (Reference, then GetValue → bare function). `.bind(this)` produces a BoundFunction with `[[BoundThis]] = this`. The assignment to `this.log` (LHS) creates an *own property* on the instance that **shadows** the prototype method.

| Aspect | Verdict |
|---|---|
| **Memory cost** | One BoundFunction per method per instance (same as arrow field) |
| **`this`-locking strength** | Strong via the BoundFunction wrapper (covered in [`call`, `apply`, `bind`](call-apply-bind.md)) |
| **Removable?** | Yes — `instance.log` is stable identity |
| **`super`-callable?** | Yes — the bound target is the prototype method, which has `[[HomeObject]]` |
| **Subclass override?** | Trickier — subclass must also bind |
| **Status** | Legacy. Predates class fields (ES2022). Use only when targeting environments without field syntax (ancient transpilers, very old browsers). For new code, **prefer arrow field** — same cost, less boilerplate. |

> **Aside —** Notice the constructor needs to repeat `this.method = this.method.bind(this)` for *every* method. With many methods this is noisy and easy to forget; the field-syntax form (arrow field) folds it into the field declaration itself. This is mostly why arrow fields won out.

### 1.5.4. `EventListener` object protocol — no extraction, no wrapper

The DOM-specific fix that was deferred from *the DOM exception* (it's a *fix*, not a `this`-contract explanation, so it belongs alongside the others here). `addEventListener` accepts two listener shapes:

```js
button.addEventListener("click", fn);                                  // function listener — extraction risk
button.addEventListener("click", obj);                                 // object listener — must have handleEvent
```

When the listener is an object, dispatch calls `obj.handleEvent(event)` — a **member-expression call** — so the Reference-base rule gives `this = obj` for free.

```js
"use strict";                                                          // L1
class Logger {                                                         // L2
  prefix = "[LOG]";                                                    // L3
  handleEvent(event) {                                                 // L4
    console.log(`${this.prefix} ${event.type}`);                       // L5 — this = the Logger instance
  }                                                                    // L6
}                                                                      // L7

const l = new Logger();                                                // L8
const button = new EventTarget();                                      // L9
button.addEventListener("click", l);                                   // L10 — pass the OBJECT, not l.handleEvent
button.dispatchEvent(new Event("click"));                              // L11 → "[LOG] click"
```

L10 is **not** an extraction — `l` is an object reference, not a member expression. The Reference-base game doesn't apply because there's no member access. Inside dispatch, `l.handleEvent(event)` is the call shape, which puts `l` on the Reference base where it should be.

| Aspect | Verdict |
|---|---|
| **Memory cost** | None per registration — reuses the existing object identity |
| **`this`-locking strength** | Strong — comes from the call shape, not a wrapper |
| **Removable?** | Yes — `removeEventListener("click", l)` works (same object identity) |
| **One method per object** | Yes — for multiple event types, switch on `event.type` inside `handleEvent`, or use multiple objects |
| **Common in the wild?** | Rare — most tutorials don't mention it |
| **Status** | DOM-only (no equivalent for setTimeout, EventEmitter, Promise, etc.). Underused but legitimate when a class is *the* listener for several events on the same target. |

When to reach for it:

- A class is dedicated to handling events on one target. Define `handleEvent(event)` with a switch on `event.type`. Single registration per event-type, no per-method binding.
- You want subscribe/unsubscribe to track *the entity* (the object), not per-method wrappers.

When not to:

- One-off listeners — overkill.
- The class needs different `this` per event (it doesn't — but if your design suggests it does, that's a refactor signal).
- Anything other than `addEventListener` — the protocol is DOM-specific.

### 1.5.5. The `removeEventListener` identity trap

A pitfall that hits *arrow at call site* and *`bind`-in-constructor* hard, *arrow field* and *`EventListener` object* not at all. Worth its own beat because it's a silent footgun.

`removeEventListener(type, listener)` matches the listener by **object identity**, not by source code or by any "deep equality" check. If the function value passed to `add` and the function value passed to `remove` aren't `===`, removal silently does nothing — no error, no warning.

```js
"use strict";                                                          // L1
const button = new EventTarget();                                      // L2
function handler() { console.log("clicked"); }                         // L3

button.addEventListener("click", handler.bind(null));                  // L4 — fresh BoundFunction
button.removeEventListener("click", handler.bind(null));               // L5 — DIFFERENT BoundFunction
button.dispatchEvent(new Event("click"));                              // L6 → still prints "clicked"
```

L4 and L5 each call `.bind` — each call produces a *new* BoundFunction object. Different identities. `removeEventListener` looks up `handler.bind(null)` from L5 in its registered-listener list, doesn't find it, returns silently. The listener registered at L4 is still there.

Same trap with arrow expressions:

```js
button.addEventListener("click", () => l.log());                       // never removable — anonymous
button.removeEventListener("click", () => l.log());                    // different arrow, different identity
```

The fix: **save the wrapper in a variable**:

```js
const handler = () => l.log();                                         // L1
button.addEventListener("click", handler);                             // L2
button.removeEventListener("click", handler);                          // L3 — same identity, works
```

Or with `bind`:

```js
this.boundLog = this.log.bind(this);                                   // L1 — store once
button.addEventListener("click", this.boundLog);                       // L2
button.removeEventListener("click", this.boundLog);                    // L3 — same wrapper, works
```

Why *arrow field* and *`EventListener` object* are immune:

- **Arrow field** — `l.log` reads the same property each time. Same identity. `add` and `remove` both reference the per-instance arrow.
- **`EventListener` object** — `l` is the same object identity each time. Trivially removable.

| Fix | Removable without explicit save? |
|---|---|
| Arrow at call site (inline) | No — anonymous arrow has no name to refer back to |
| Arrow field | Yes — instance property is stable identity |
| `bind`-in-constructor | Yes — assigned to `this.method`, stable identity |
| Bare `.bind` at registration | No — each `.bind` makes a new wrapper |
| `EventListener` object | Yes — object identity is stable |

> **Aside —** `addEventListener` does *one* helpful thing: registering the same listener twice for the same event-type/capture-phase is a no-op (the second call is silently dropped). So you can't accidentally double-fire by re-registering. But you can't accidentally `bind`-equal either, and that's where most "I called remove but it still fires" bugs come from.

### 1.5.6. `Function.prototype.bind` outside the constructor — the bare `.bind` pattern

For completeness, the most common form of `bind` use in real code: inline at the registration site.

```js
button.addEventListener("click", l.log.bind(l));                       // L1 — bind at the call site
```

Mechanism: `l.log.bind(l)` produces a fresh BoundFunction with `[[BoundThis]] = l`. The dispatcher calls the BoundFunction; the wrapper substitutes `l` for whatever `thisValue` the dispatcher passed. Inside `log`, `this = l`.

| Aspect | Verdict |
|---|---|
| **Memory cost** | One BoundFunction per registration |
| **`this`-locking strength** | Strong — same as bind-in-constructor |
| **Removable?** | **No** — anonymous BoundFunction. Save it: `const bound = l.log.bind(l); add(bound); ... remove(bound);` |
| **Verbosity** | Repeats `l` twice; slightly noisier than arrow at call site |
| **Status** | Legitimate but largely superseded by *arrow at call site*, which is more readable for the "wrap the dispatcher's call shape" task. Use `bind` here only when you also want partial application of arguments — a real `bind` superpower arrows-at-call-site can't match. |

```js
button.addEventListener("click", l.log.bind(l, "[manual prefix]"));    // partial application — bind shines
```

### 1.5.7. Summary — five fixes, one decision

| Mechanism | Inserts what? | Memory | Removable? | When |
|---|---|---|---|---|
| Arrow at call site | An arrow that does the call | 1 per registration | Only if saved to var | One-off registrations; dispatcher arg shape ≠ method arg shape |
| Arrow field | A locked-`this` arrow as instance property | 1 per instance per method | Yes | Methods primarily passed as callbacks; you control the class |
| `bind`-in-constructor | A BoundFunction shadowing the prototype method | 1 per instance per method | Yes | Legacy; prefer arrow field |
| `EventListener` object | Nothing (uses member-expression call) | 0 | Yes | DOM-only, class is *the* listener |
| Bare `.bind` | A BoundFunction at the call site | 1 per registration | Only if saved to var | Partial application is wanted alongside `this`-locking |

> **Aside —** Stage-3 decorators (TC39, partially landed in TS 5.0+) include `@bound` and similar primitives that compile to one of these patterns. They don't introduce a *new* mechanism — they're sugar over arrow field / bind-in-constructor. Status: not yet baseline in plain JS. Use them only behind a transpiler. Once they reach Stage 4 and bake into engines, they become the cleanest option for "this method is always self-bound."
