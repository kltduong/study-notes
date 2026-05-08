# EventTarget and the Web's Event System

**TL;DR**

- `EventTarget` is the base interface for **anything** that can fire / listen for events — not just DOM nodes.
- It sits one level above `Node` in the hierarchy specifically because **many event-emitting things are not DOM nodes** (`Window`, `XMLHttpRequest`, `WebSocket`, `AbortSignal`, workers, …).
- It's directly constructible since 2021 — useful for building your own event-emitting objects without inheriting from `Node`.

---

## Why `EventTarget` exists above `Node`

Looking at the DOM hierarchy, it might seem `EventTarget` is a useless extra layer when only `Node` extends it:

```bash
EventTarget
└── Node
    └── Element
        └── ...
```

But the full picture is much wider — many event-emitting things are **not** DOM nodes:

```bash
EventTarget
├── Node                 (DOM tree stuff)
├── Window               ← not a Node!
├── XMLHttpRequest
├── AbortSignal          (from AbortController)
├── WebSocket
├── EventSource          (server-sent events)
├── MessagePort          (postMessage channels)
├── Worker / ServiceWorker / SharedWorker
├── BroadcastChannel
├── MediaQueryList       (window.matchMedia(...))
├── IDBRequest           (IndexedDB)
├── Notification
├── PerformanceObserver
├── RTCPeerConnection
└── ...many more
```

If `addEventListener` lived on `Node` instead, every one of these would have to reinvent its own event API.

This is **interface segregation**: "can receive events" (`EventTarget`) is separated from "is in the DOM tree" (`Node`). A `Window` _is_ an event target but _isn't_ a tree node — and the type system reflects that honestly.

## Examples — non-Node event targets

```js
// Window — not a Node
window.addEventListener("resize", ...);

// XHR — not a Node
const xhr = new XMLHttpRequest();
xhr.addEventListener("load", ...);

// AbortSignal — not a Node
const ctrl = new AbortController();
ctrl.signal.addEventListener("abort", ...);

// WebSocket — not a Node
const ws = new WebSocket(url);
ws.addEventListener("message", ...);
```

## Constructible directly

Since 2021 (modern browsers), you can `new EventTarget()` or extend it — handy for custom event buses without inheriting DOM machinery:

```js
class Bus extends EventTarget {}
const bus = new Bus();
bus.addEventListener("ping", (e) => console.log(e.detail));
bus.dispatchEvent(new CustomEvent("ping", { detail: 42 }));
```

## The EventTarget interface (just three methods)

```js
target.addEventListener(type, listener, options?);
target.removeEventListener(type, listener, options?);
target.dispatchEvent(event);   // returns false if cancelled
```

That's it. Everything else (capture/bubble phases, `event.target` vs `event.currentTarget`, etc.) is built on top.

---

## Related

- [DOM: Node, Element, and Collections](./dom-collections.md) — the DOM-tree side of the hierarchy.
