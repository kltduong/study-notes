# JS: Own Properties vs Prototype (and how Python compares)

**TL;DR**
- Methods declared in a `class` body live on `Class.prototype` — one shared copy for all instances.
- Things assigned via `this.x = ...` (and class fields like `x = 1`) become **own properties** of the instance.
- Method lookup walks the prototype chain: `obj → obj.__proto__ → ... → null`.
- Python has the same conceptual split (instance attrs vs class attrs), but with two big differences:
  1. `class Foo: x = []` in Python is a **shared class attribute** (mutable-default trap). In JS, `class Foo { x = [] }` is a **per-instance field**.
  2. Python auto-binds `self`; JS does not auto-bind `this`.

---

## The rule, by syntax

```js
class Foo {
  constructor() {
    this.x = 1;             // own property
    this.greet = () => {};  // own property
  }

  bar() {}                  // Foo.prototype.bar
  get y() { return 2; }     // getter on Foo.prototype
  static baz() {}           // Foo.baz (on the class itself)
}

const f = new Foo();
f.hasOwnProperty('x')       // true
f.hasOwnProperty('bar')     // false → on Foo.prototype
```

| Lives as **own property** | Lives on **prototype** |
|---|---|
| `this.x = ...` in constructor | Methods declared in class body |
| Class fields: `class Foo { x = 1 }` | Getters/setters declared in class body |
| Properties added later: `obj.foo = 1` | `constructor` itself |
| Array elements, Map/Set entries | Inherited methods from parent classes |

> **Class fields are sugar for `this.x = ...` in the constructor** — so they're own properties, *not* on the prototype. Easy to misread.

## Why this split

- **Shared behavior** (methods, getters) → prototype. One function shared by all instances.
- **Per-instance state** → own properties. Each instance's value differs.
- **Memory:** without prototypes, every DOM node would carry its own copy of `appendChild`.
- **Patchability:** monkey-patching `Document.prototype.createElement` affects every document.

## DOM example

```js
const div = document.createElement('div');

Object.keys(div)   // []  ← no own enumerable properties
```

`div.id` looks like a normal property, but it's actually a **getter/setter on `Element.prototype`** that proxies the underlying attribute. Almost everything on DOM objects lives on prototypes:

```
HTMLDivElement.prototype  → (mostly empty)
HTMLElement.prototype     → .click(), .focus(), .dataset, .style
Element.prototype         → .id, .className, .classList, .getAttribute, .querySelector
Node.prototype            → .appendChild, .childNodes, .parentNode
EventTarget.prototype     → .addEventListener, .dispatchEvent
```

That's why `document.createElement` isn't an own property of `document` — it lives on `Document.prototype` and is found via chain lookup.

## Inspecting any object

```js
Object.getOwnPropertyNames(obj)        // own (incl. non-enumerable)
Object.keys(obj)                       // own enumerable only
Object.getPrototypeOf(obj)             // → the prototype

// Walk the whole chain:
let p = obj;
while (p) {
  console.log(p.constructor?.name, Object.getOwnPropertyNames(p));
  p = Object.getPrototypeOf(p);
}
```

> Prototype methods are **non-enumerable** by default — `Object.keys(Cls.prototype)` returns `[]`. Use `getOwnPropertyNames` to see them.

---

## JavaScript vs Python

Same conceptual split, different mechanism.

| JavaScript | Python |
|---|---|
| Own property | Instance attribute (`self.__dict__`) |
| Prototype property | Class attribute (`Cls.__dict__`) |
| `Object.getPrototypeOf(obj)` | `type(obj)` then walks `Cls.__mro__` |
| Prototype chain (objects) | MRO (classes, with C3 linearization for multiple inheritance) |

### The big trap: class-body field declarations

```python
class Foo:
    x = []          # CLASS attribute — SHARED across all instances!

a, b = Foo(), Foo()
a.x.append(1)
b.x   # [1]  ← surprise
```

```js
class Foo {
  x = [];   // INSTANCE field — fresh array per instance
}
const a = new Foo(), b = new Foo();
a.x.push(1);
b.x;   // []
```

In Python you must do `self.x = []` in `__init__` for per-instance state. The mutable-default-on-the-class footgun bites Python beginners constantly.

### Assignment shadows, doesn't mutate (both languages)

Writing `obj.attr = ...` always creates an attribute on the **instance**, never on the class/prototype.

```python
class Foo: x = 1
f = Foo()
f.x = 99           # creates instance attribute, shadows class attribute
Foo.x              # still 1
del f.x; f.x       # 1 (back to class attribute)
```

```js
class Foo {}
Foo.prototype.x = 1;
const f = new Foo();
f.x = 99;          // own property, shadows prototype
Foo.prototype.x;   // still 1
delete f.x; f.x;   // 1
```

### Lookup chain shape

- **JS:** chain of *objects* (`obj → __proto__ → ... → null`). Single inheritance at the prototype level.
- **Python:** chain of *classes* via MRO. Genuine multiple inheritance, linearized by C3.

### Method binding

- **Python:** `obj.method` returns a *bound method* — `self` is auto-bound. Storing and calling later just works.
- **JS:** `obj.method` returns the raw function. `this` is decided at call site.

```python
m = counter.increment
m()   # works, self is bound
```

```js
const m = counter.increment;
m();  // TypeError or wrong `this`
// fix: counter.increment.bind(counter), or arrow methods
```

### Static / class-level

| Goal | JS | Python |
|---|---|---|
| Per-class shared method | `static foo()` | `@staticmethod` / `@classmethod` |
| Per-class shared data | `Foo.x = 1` or `static x = 1` | `class Foo: x = 1` (the natural default!) |

### Inspection cheatsheet

| Goal | JS | Python |
|---|---|---|
| Own/instance attrs | `Object.getOwnPropertyNames(obj)` | `obj.__dict__` |
| Class/proto attrs | `Object.getOwnPropertyNames(Cls.prototype)` | `Cls.__dict__` |
| Walk the chain | `Object.getPrototypeOf` repeatedly | `type(obj).__mro__` |

---

## Related

- [DOM: Node, Element, and Collections](./dom-node-element-collections.md) — concrete example of how DOM relies entirely on prototype methods.
