# DOM: Node, Element, and Collections

**TL;DR**

- `Node` is the base class; `Element` is one subtype. Text and comments are nodes but **not** elements.
- `*Element*`-named traversal APIs (`children`, `firstElementChild`, `nextElementSibling`, …) skip text/comment nodes; the `*Node*` / generic ones (`childNodes`, `firstChild`, `nextSibling`, …) don't.
- `HTMLCollection` is element-only and **always live**. `NodeList` can hold any node and is **usually static** (except `childNodes`).
- `document` (lowercase) is the singleton instance; `Document` (capital) is the class. Other `Document` instances exist (iframes, `DOMParser`).

---

## Class hierarchy

```bash
EventTarget
├── Node                       ← the DOM tree (shown below)
│   ├── Element                (e.g. <div>, <p>, <a>)
│   │   └── HTMLElement
│   │       ├── HTMLDivElement
│   │       └── ...
│   ├── Text                   (text between tags)
│   ├── Comment                (<!-- ... -->)
│   ├── Document               (the whole document)
│   │   └── HTMLDocument       ← what `document` actually is
│   ├── DocumentFragment       (lightweight container, e.g. shadow roots)
│   └── Attr
└── (also: Window, XMLHttpRequest, WebSocket, AbortSignal, Worker, ...)
```

> `EventTarget` is **not DOM-specific** — many non-Node things (`Window`, `XHR`, `WebSocket`, etc.) extend it directly. See [EventTarget and the Web's Event System](./event-target-and-events.md) for the full picture. The rest of this note focuses on the `Node` subtree.

Every `Element` is a `Node`, but a `Text` node (the actual characters inside `<p>hello</p>`) is a `Node` and **not** an `Element`.

**Why the split?** A document isn't a tree of tags — it's a tree of heterogeneous things. Text, comments, and the document itself all participate in the tree (parent/child relationships, insertion, removal), so they need the shared `Node` behavior. But only real tags have tag names, attributes, classes, and CSS — that's what `Element` adds. Separating the two keeps `Node` honest (text really doesn't have a `className`) and lets APIs opt in to the narrower type.

## Node vs Element — traversal gotcha

```js
const p = document.querySelector("p"); // <p>hello <b>world</b></p>

p.children; // HTMLCollection — only Elements: [<b>]
p.childNodes; // NodeList — all Nodes: [#text "hello ", <b>]

p.firstChild; // Text node "hello "
p.firstElementChild; // <b>
```

The `*Element*` variants skip text and comment nodes. Forgetting this is a classic source of bugs when `firstChild` returns whitespace text instead of the element you wanted.

**Why two sets of APIs?** Some code cares about the literal DOM (e.g. a serializer, or a text editor preserving whitespace) — it needs to see every node. Most app code only cares about structural tags and treats inter-tag whitespace as noise. Rather than force one camp to filter on every access, the spec provides both. The naming convention (`*Element*` vs `*Node*` / generic) is the signal: if the name says "Element," text and comments are hidden.

---

## HTMLCollection vs NodeList

**Why two collection types at all?** Historical accident + type narrowing. `HTMLCollection` predates `NodeList` and was designed for element-only, always-live views (`document.forms`, `document.images`, `.children`). `NodeList` came later to handle the general case (any node, sometimes static). The web platform never consolidates — it accretes — so both stayed, and APIs return whichever matches their legacy contract.

|                               | `HTMLCollection`  | `NodeList`                               |
| ----------------------------- | ----------------- | ---------------------------------------- |
| Contains                      | Elements only     | Any Node                                 |
| Live?                         | Always live       | Usually static (exception: `childNodes`) |
| Access by name                | Yes (`coll.myId`) | No                                       |
| `forEach`                     | No                | Yes                                      |
| Indexed access                | Yes               | Yes                                      |
| Iterable (`for...of`, spread) | Yes               | Yes                                      |

### Live vs static

A **live** collection re-queries the DOM on every read. **Static** is a snapshot.

**Why would you ever want live?** Because sometimes a view *should* track reality — a sidebar counter showing `.unread-message` count should update automatically as new messages arrive, without you re-querying. Live collections were the original design: the DOM is mutable, so collections reflecting it felt natural. The hidden cost is the footgun below (mutation-during-iteration) and a potential perf cliff on large documents. `querySelectorAll` was added later precisely because most modern code wants the predictable snapshot semantics.

```js
const live = document.getElementsByTagName("div"); // HTMLCollection — live
const stat = document.querySelectorAll("div"); // NodeList — static

document.body.appendChild(document.createElement("div"));

live.length; // increased by 1
stat.length; // unchanged
```

**Which APIs return what:**

| API                                                           | Returns                             |
| ------------------------------------------------------------- | ----------------------------------- |
| `getElementsByTagName`, `getElementsByClassName`, `.children` | live `HTMLCollection`               |
| `querySelectorAll`                                            | static `NodeList`                   |
| `childNodes`                                                  | live `NodeList` (the rare live one) |

### Gotchas

**1. Mutating a live collection while iterating** — items drop out, indices skip:

```js
const items = document.getElementsByClassName("todo");
for (let i = 0; i < items.length; i++) {
  items[i].classList.remove("todo"); // item leaves the collection
  // next iteration skips an element
}
```

Fix: iterate backwards, snapshot via `[...items]`, or use `querySelectorAll`.

**2. `forEach` only exists on `NodeList`:**

```js
document.querySelectorAll('div').forEach(...)        // works
document.getElementsByTagName('div').forEach(...)    // TypeError
[...document.getElementsByTagName('div')].forEach(...)  // works
```

**3. Neither is a real `Array`** — no `.map`, `.filter`, `.reduce`. Use spread or `Array.from` first.

### Rule of thumb

Default to `querySelectorAll`: static + has `forEach` + flexible selectors. Reach for `.children` / `getElementsByClassName` only when you specifically want the live behavior (e.g. an auto-updating counter).

---

## document vs Document

- **`Document`** (capital) — the **class/interface**. Used in `instanceof` checks and TypeScript types.
- **`document`** (lowercase) — the **singleton instance** for the current page, hanging off the global.

**Why both?** The class has to exist (the DOM is object-oriented, and documents have behavior — `createElement`, `querySelector`, etc. — that belongs on a type). A *convenience global* has to exist too, because 99% of code is talking about *this* page's document and nobody wants to type `window.currentDocument()` every time. So the platform exposes a pre-created singleton pointing at the current page. Any time you see a `foo` / `Foo` pair on `window`, this is the pattern.

```js
document instanceof Document; // true
document.constructor === HTMLDocument; // true (HTMLDocument extends Document)
```

### Other Document instances exist

`document` isn't unique — you can create or get others:

```js
// Parse a string
const doc = new DOMParser().parseFromString("<p>hi</p>", "text/html");
doc instanceof Document; // true
doc === document; // false

// An <iframe> has its own Document
const iframeDoc = document.querySelector("iframe").contentDocument;
iframeDoc === document; // false
```

### Cross-document moves need adoption

Nodes are **owned by a document** (`node.ownerDocument`). Moving a node between documents:

**Why ownership matters.** Each `Document` is its own universe: it has its own prototype instances, its own stylesheet references, and — crucially for iframes — potentially a different **origin** with different security boundaries. A node carries a back-pointer to its owner so the engine knows which universe's rules apply. Naively grafting a node from iframe A into iframe B would leave it pointing at A's types and A's origin while living inside B's tree, which is a mess. `importNode` / `adoptNode` explicitly rebind ownership so the engine can rewire the node cleanly.

```js
const node = iframeDoc.createElement("div");
// Historically threw WRONG_DOCUMENT_ERR. Safer:
document.body.appendChild(document.importNode(node, true));
// or document.adoptNode(node)
```

### The capital/lowercase pattern generalizes

| Instance (lowercase) | Class (capital)             |
| -------------------- | --------------------------- |
| `window`             | `Window`                    |
| `document`           | `Document` (`HTMLDocument`) |
| `navigator`          | `Navigator`                 |
| `location`           | `Location`                  |
| `history`            | `History`                   |

Lowercase = pre-created singleton on the global. Capital = the type.

---

## Related

- [JS prototype vs own properties](./prototype-vs-own-properties.md) — why `document.createElement` lives on `Document.prototype`, not on `document` itself.
