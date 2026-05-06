# Client-Side JavaScript

Notes on how JavaScript runs in the browser: the platform layers beneath the language, the object model the DOM is built on, the event system the web is glued together with, and the rendering pipeline that JS executes inside.

## The mental model

Client-side JS sits on top of several layers that are easy to conflate:

- **The language** (ECMAScript) — just syntax, types, and pure built-ins like `Array` and `Promise`. No `document`, no `fetch`.
- **The host** (the browser) — adds the DOM, network, storage, timers, and the event loop. This is what makes JS useful for web pages.
- **The engine's object representation** — the _how_ behind the object model. Objects behave like hash maps but are implemented as shared schemas (shapes) + value slots, with inline caches on hot property-access sites. The spec doesn't mandate this, but every major engine does it.
- **The object model** — everything the host exposes is wired into JS via prototype chains. `document.createElement` isn't a property of `document`; it lives on `Document.prototype`. DOM APIs feel like language features because prototypes blur the line.
- **Events** — the web's pub/sub glue. `EventTarget` is a separate interface specifically because many event-emitting things (`Window`, `XHR`, `WebSocket`, `AbortSignal`) are not DOM nodes.

A lot of "why does this API look weird?" moments resolve once you know which layer owns the thing you're looking at.

## Reading order

Start with the platform layers, then the object model (semantics + implementation), then the DOM, then events. Each note stands alone, but they reinforce each other in this order.

1. **[js-engine-runtime.md](./js-engine-runtime.md)** — ECMAScript vs JavaScript vs engine vs runtime. What's in the engine (pure language) vs what the host adds (`fetch`, `window`, `fs`, …), and how the runtime registers native functions into the engine's global. The layer map for everything below.
2. **[js-inheritance.md](./js-inheritance.md)** — the prototype chain mechanism, the four direct ways to set `[[Prototype]]`, how `class`/`new` build on it, own vs prototype properties, `this` resolved at the call site, statics, and a JS-vs-Python comparison. The object-model primer.
3. **[shapes-inline-caches.md](./shapes-inline-caches.md)** — how engines actually store objects. Shape = shared schema; inline cache = remembered offset at a call site. Explains why JS is fast despite semantically being hash maps, and why property order / `delete` / optional fields matter for performance.
4. **[dom-collections.md](./dom-collections.md)** — the `Node`/`Element` class hierarchy, `childNodes` vs `children`, and `HTMLCollection` (live) vs `NodeList` (usually static). What the browser runtime actually puts in front of you as "the DOM."
5. **[events-targets.md](./events-targets.md)** — why `EventTarget` is its own interface above `Node`, and all the non-Node things that also fire events.
6. **[event-dispatch.md](./event-dispatch.md)** — what actually happens inside event dispatch: tasks as jobs (not callbacks), the native→JS boundary, physical vs scripted dispatch, and a map for going deeper (WHATWG spec, DOM dispatch algorithm, V8 internals, Scheduler API).
7. **[rendering-path.md](./rendering-path.md)** — the Critical Rendering Path: how the browser turns HTML/CSS/JS into pixels (DOM → CSSOM → Render Tree → Layout → Paint → Composite), and what blocks what along the way. Where `async`, `defer`, `preload`, and critical CSS fit in.

## Cross-cutting ideas

### "Where does this API live?"

A useful first question whenever something behaves surprisingly across environments:

| Layer                 | Owns         | Examples                                           |
| --------------------- | ------------ | -------------------------------------------------- |
| ECMAScript (engine)   | The language | `Array`, `Promise`, `JSON`, `Map`, syntax          |
| Browser runtime       | Host APIs    | `document`, `fetch`, `setTimeout`, `localStorage`  |
| `Node.prototype` etc. | DOM behavior | `appendChild`, `addEventListener`, `querySelector` |

If an API only works in one runtime, it's almost always the runtime row. If a method is missing on an object you expected it on, it's almost always the prototype row (walk the chain).

### Prototypes are the connective tissue

`document.querySelector`, `element.addEventListener`, `node.appendChild` — none of these are own properties of the objects they appear on. They live on `Document.prototype`, `EventTarget.prototype`, `Node.prototype`. The inheritance note covers the chain mechanism and how classes build on it; the DOM note shows it in practice — once you can walk a prototype chain, DOM APIs stop looking magical.

### Semantics vs representation

`js-inheritance.md` and `shapes-inline-caches.md` are two views of the same objects. The first is what the language _promises_ (own vs prototype, dynamic property addition, `this` at the call site). The second is how the engine _delivers_ on those promises quickly (shared shapes, inline caches, dictionary-mode fallback). The inheritance note tells you what you can do; the shapes note tells you which of those things are cheap.

### `EventTarget` vs `Node`

A recurring theme: the web platform segregates interfaces by capability, not by category. `EventTarget` is above `Node` because "can receive events" is a broader capability than "is a DOM tree node." `Window` is an event target but not a node; a `Text` node is a node but not an element. The hierarchy is honest about this.

## Related topics

- [css/](../css/) — selectors that target the DOM described here.
