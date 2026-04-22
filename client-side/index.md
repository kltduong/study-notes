# Client-Side JavaScript

Notes on how JavaScript runs in the browser: the platform layers beneath the language, the object model the DOM is built on, and the event system the web is glued together with.

## The mental model

Client-side JS sits on top of several layers that are easy to conflate:

- **The language** (ECMAScript) — just syntax, types, and pure built-ins like `Array` and `Promise`. No `document`, no `fetch`.
- **The host** (the browser) — adds the DOM, network, storage, timers, and the event loop. This is what makes JS useful for web pages.
- **The object model** — everything the host exposes is wired into JS via prototype chains. `document.createElement` isn't a property of `document`; it lives on `Document.prototype`. DOM APIs feel like language features because prototypes blur the line.
- **Events** — the web's pub/sub glue. `EventTarget` is a separate interface specifically because many event-emitting things (`Window`, `XHR`, `WebSocket`, `AbortSignal`) are not DOM nodes.

A lot of "why does this API look weird?" moments resolve once you know which layer owns the thing you're looking at.

## Reading order

Start with the platform layers, then the object model, then the DOM, then events. Each note stands alone, but they reinforce each other in this order.

1. **[ecmascript-engine-runtime.md](./ecmascript-engine-runtime.md)** — ECMAScript vs JavaScript vs engine vs runtime. What's in the engine (pure language) vs what the host adds (`fetch`, `window`, `fs`, …), and how the runtime registers native functions into the engine's global. The layer map for everything below.
2. **[js-class-semantics.md](./js-class-semantics.md)** — own properties vs prototype properties, how `this` is resolved at the call site, and how this differs from Python's bound methods. The object-model primer.
3. **[dom-node-element-collections.md](./dom-node-element-collections.md)** — the `Node`/`Element` class hierarchy, `childNodes` vs `children`, and `HTMLCollection` (live) vs `NodeList` (usually static). What the browser runtime actually puts in front of you as "the DOM."
4. **[event-target-and-events.md](./event-target-and-events.md)** — why `EventTarget` is its own interface above `Node`, and all the non-Node things that also fire events.

## Cross-cutting ideas

### "Where does this API live?"

A useful first question whenever something behaves surprisingly across environments:

| Layer | Owns | Examples |
|---|---|---|
| ECMAScript (engine) | The language | `Array`, `Promise`, `JSON`, `Map`, syntax |
| Browser runtime | Host APIs | `document`, `fetch`, `setTimeout`, `localStorage` |
| `Node.prototype` etc. | DOM behavior | `appendChild`, `addEventListener`, `querySelector` |

If an API only works in one runtime, it's almost always the runtime row. If a method is missing on an object you expected it on, it's almost always the prototype row (walk the chain).

### Prototypes are the connective tissue

`document.querySelector`, `element.addEventListener`, `node.appendChild` — none of these are own properties of the objects they appear on. They live on `Document.prototype`, `EventTarget.prototype`, `Node.prototype`. The class-semantics note and the DOM note reinforce each other: once you can walk a prototype chain, DOM APIs stop looking magical.

### `EventTarget` vs `Node`

A recurring theme: the web platform segregates interfaces by capability, not by category. `EventTarget` is above `Node` because "can receive events" is a broader capability than "is a DOM tree node." `Window` is an event target but not a node; a `Text` node is a node but not an element. The hierarchy is honest about this.

## Related topics

- [css/](../css/) — selectors that target the DOM described here.
