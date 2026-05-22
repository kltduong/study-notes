# Patterns & Pitfalls — teaching draft

## Plan (teaching order)

- [x] **0. Teaser** — DOM `addEventListener` extraction; surprising `this` (not `undefined`)
- [x] **1. The pitfall sites** — catalog of where extraction happens (callbacks, event handlers, setTimeout, destructuring, higher-order)
- [x] **2. The DOM exception** — Web IDL "invoke a callback" sets `this = currentTarget`; consequences
- [ ] **3. The fix spectrum** — arrow at call site, arrow field, `bind`-in-constructor, `EventListener` object protocol; status/when to use
- [ ] **4. The `removeEventListener` identity trap** — wrappers break removal symmetry
- [ ] **5. Decision matrix** — pulling §1–§4 into a "which fix for which site" table
- [ ] **6. Worked synthesis** — one example showing 3+ sites + 3+ fixes side-by-side

## 0. Teaser

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


### 0.1. Reveal

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

#### What dispatch actually does

The DOM standard's event dispatch algorithm calls the listener through Web IDL's **"invoke a callback function"** operation. The relevant step (paraphrased):

> Call the callback's `[[Call]]` internal method with **`thisValue = the EventTarget being dispatched on`** and the event arguments.

Effectively, `target.dispatchEvent(evt)` calls `listener.[[Call]](target, [evt])` — equivalent to `listener.call(target, evt)`, not `listener(evt)`. The `addEventListener` API specifies `currentTarget` as the `this` value for the callback.

So in our snippet:

```js
button.addEventListener("click", l.log);   // L8 — extraction (Reference discarded), bare function stored
button.dispatchEvent(new Event("click"));  // L9 — calls l.log.[[Call]](button, [event])
```

Inside `log`, `this = button` (an `EventTarget`). `button.prefix` doesn't exist → reads as `undefined` — no error, just a missing property. Template literal interpolates `undefined` to the string `"undefined"`.

#### Why this matters

The extraction *did* happen. The Reference was lost. But the next call site **isn't a plain call** — it's a DOM-driven call with a built-in `this` override. So the failure mode is different:

| Caller of extracted function | What it does | `this` inside method | Symptom |
|---|---|---|---|
| Plain call (`fn()`, `setTimeout(fn, 0)`, `forEach(fn)`) | Identifier-based `[[Call]]`, no override | `undefined` (strict) | TypeError on first property read |
| DOM event dispatch | Web IDL invoke with `thisValue = currentTarget` | The EventTarget | **Silent** wrong-object behavior — undefined property reads, no errors |
| Native `bind`-style dispatchers (rare) | Varies by API contract | API-specific | Varies |

DOM events are the dominant case where extraction produces a *silent* bug instead of a loud TypeError. That's what makes them especially nasty:

- Loud failure (`fn()` → TypeError) tells you *immediately* that something's wrong.
- Silent failure (`addEventListener` → `target.prefix → undefined`) means the handler "runs" — and your debugging starts with "why isn't the counter incrementing?" rather than "where's the TypeError?".

#### Tying it back to the pipeline

This is consistent with everything we've built. `dispatchEvent` is just another caller; the only question is "what does the caller pass to `[[Call]]` as `thisValue`?":

- Reference-base call site → uses `[[Base]]`
- Plain identifier call site → ER → strict-`undefined` (or sloppy-`globalThis`)
- `func.call(arg)` / `func.apply(arg, ...)` → uses `arg`
- BoundFunction wrapper → uses `[[BoundThis]]`, ignores incoming
- **DOM event dispatch → uses `currentTarget`** (specified by the DOM standard, not ECMAScript)

The mechanism itself is unchanged — only the source of `thisValue` varies. Adding "DOM dispatch" to the list of `thisValue` suppliers is the small addition. The big lesson is the *failure-mode shift*: silent vs loud.



## 1. The pitfall sites

§0 showed one site (DOM dispatch). The full catalog: every call site that takes a function value, stores it, and invokes it later through *some* dispatcher. The dispatcher decides what `this` is — your code has no say once the function is handed over.

Three things have to happen for a `this`-loss bug:

1. **Extraction** — a member expression is evaluated in a position that requires `GetValue` (argument, assignment, destructuring, return). The Reference is discarded; only the bare function object survives.
2. **Storage** — the function is held somewhere (parameter slot, array, listener registry, timer queue, microtask queue).
3. **Re-invocation** — a dispatcher later calls the function via *its own* `thisValue` choice — not the original object.

Steps 1 and 3 can be far apart in the source — that's what makes these bugs slippery.

### 1.1. Extraction shapes

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
| `obj.method?.()` | Optional-call preserves Reference (verified in chunk 3) |
| `(obj.method)()` | Parentheses around a Reference don't trigger GetValue |
| `obj.method.bind(obj)` | `.bind` is invoked *as a method on the function*, then the BoundFunction stores `obj` |

### 1.2. Re-invocation sites — what each dispatcher passes as `this`

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
| `EventTarget.dispatchEvent` (via `addEventListener`) | `currentTarget` | **Silent wrong-object** (§0) |
| `node.on("event", fn)` (Node EventEmitter) | The `EventEmitter` instance | Silent wrong-object |
| Web Worker `postMessage` handler | The `Worker`/global scope | Silent wrong-object |
| `MutationObserver` callback | The observer instance | Silent wrong-object |
| Constructor (`new fn(...)`) | A fresh object (then return-value override applies) | Probably TypeError, possibly silent if body tolerates fresh `{}` |
| Generator `next(arg)` | The generator object | Silent wrong-object |

The pattern: **runtime-internal dispatchers (Promise, microtask, array methods)** stay close to the spec's "plain call" default and pass `undefined`. **Host-environment dispatchers (DOM, Node EventEmitter, timers)** typically supply something — usually whatever the environment thinks is the "natural target." That something is almost never the object the method came from.

### 1.3. The two failure shapes

This is the practical takeaway from the catalog:

| Shape | When | Symptom | Debug experience |
|---|---|---|---|
| **Loud failure** | Plain calls, array methods, Promise/microtask | TypeError on first `this.X` read | Stack trace points at the method body — relatively easy |
| **Silent failure** | DOM events, EventEmitter, host timers, observers | `this.X` reads `undefined`; no error; downstream logic gets garbage | Bug surfaces far from the cause — counter doesn't tick, state doesn't update, render is wrong |

Strict mode (mandatory in classes/modules) made the **loud** path louder — a property access on `undefined` throws immediately rather than reading from `globalThis`. But strict mode does nothing for the **silent** path: the dispatcher already supplied a `this`, so `this` is a real object — properties just don't exist on it.

So: arrows / `bind` / etc. exist not just to "preserve `this` for safety" — they exist to flip silent bugs into correct behavior, and (where a fix is missing) to flip silent bugs into loud ones.

> **Aside —** Several of the entries in §1.2 are themselves subject to spec evolution. Pre-ES2015, `Array.prototype.forEach`'s `this` for the callback was sloppy-`globalThis`; under strict it's `undefined`. The DOM's "use `currentTarget`" rule has been there since DOM Level 2 and isn't going to change, but the rule for *new* host APIs ("what `this` should we pass?") is still made case-by-case by spec authors. The mental model — "find the dispatcher, find its contract" — generalizes; the table doesn't.


## 2. The DOM exception — why dispatch passes `currentTarget`

§0 stated the rule: DOM event dispatch calls listeners with `this = currentTarget`. This section unpacks *why* — both the spec mechanism and the historical motivation. Once both layers are clear, the silent-failure pattern stops looking arbitrary.

### 2.1. Two specs, two layers

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

### 2.2. Why `currentTarget`? (motivation)

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

In **modern** code with classes, that convention reverses: `this` "should" be the controller object that owns state, not the DOM node. So the DOM's well-meaning convenience has become the most common silent-`this`-loss source in app code.

The behavior is correct per spec; what changed is what programmers want `this` to mean. **Status:** legacy convention. Don't write new code that relies on `this = currentTarget` inside a listener — use `event.currentTarget` explicitly when you need the element, and `this` for your controller object (with one of the §3 fixes).

### 2.3. The `EventListener` object protocol — the alternate API

`addEventListener` accepts two listener shapes — most code uses the first:

```js
button.addEventListener("click", fn);                                  // function listener
button.addEventListener("click", obj);                                 // object listener — must have a handleEvent property
```

The object form is rarely shown in tutorials but has been part of DOM Level 2 (1999) since launch. When the listener is an object, dispatch calls `obj.handleEvent(event)` — a **member-expression call** — so `this` follows the standard Reference-base rule: `this = obj`.

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

Compare with the broken version from §0:

```js
button.addEventListener("click", l.log);          // extraction → DOM picks currentTarget for this
button.addEventListener("click", l);              // no extraction → DOM does l.handleEvent(event) → this = l
```

Why this works: the dispatcher's call is `listener.handleEvent(event)`. That **is** a member expression — Reference base = `listener`. Standard Reference-base rule does the right thing without any `bind`/arrow plumbing.

### 2.4. The `EventListener` object — status

| Aspect | Verdict |
|---|---|
| **Spec coverage** | Always — DOM Level 2, every browser, since launch |
| **Handles `this` correctly?** | Yes — structurally, no plumbing |
| **Memory cost** | One object per listener (vs one bound function per listener) |
| **`removeEventListener` works?** | Yes — same object identity, easily kept |
| **Common in the wild?** | Rare — most tutorials don't mention it |
| **Status** | Underused. Niche but legitimate — particularly clean for classes that act as event controllers |

When to reach for it:

- A class is *the* listener for several events. Define `handleEvent(event)` and dispatch by `event.type` inside. One registration per event, no per-method binding.
- You want to subscribe/unsubscribe a logical entity (the object) rather than tracking a wrapper per method.

When not to:

- One-off listeners or anonymous handlers — overkill.
- You need different `this` for different events on the same target — the object form gives you one `this` for all.

> **Aside —** This is the "right tool for the job" check applied to event handling: arrow field, `bind` in constructor, and `EventListener` object are three legitimate fixes. They're not interchangeable — each has different memory cost, different removability, and different expressiveness. §3 lays out the spectrum.

### 2.5. Other host dispatchers follow the same template

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
