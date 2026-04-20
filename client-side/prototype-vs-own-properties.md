# JS: Own Properties vs Prototype (and how Python compares)

**TL;DR**
- Methods declared in a `class` body live on `Class.prototype` — one shared copy for all instances.
- Properties declared in a `class` body (fields like `x = 1`) — and anything assigned via `this.x = ...` — become **own properties** of the instance.
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

Core difference: **Python binds `self` at attribute access; JS resolves `this` at call site.**

- **Python:** `obj.method` returns a *bound method* object — `self` is pre-packaged via the descriptor protocol. Storing and calling later just works.
- **JS:** `obj.method` returns the raw function. `this` is decided by **how** you call it.

```python
m = counter.increment
m()                           # works, self is bound
counter.increment.__self__    # → counter
counter.increment.__func__    # → the underlying function
```

```js
const m = counter.increment;
m();                              // TypeError or wrong `this`
counter.increment.call(counter);  // works
setTimeout(counter.increment.bind(counter), 0);  // classic fix
```

#### How `this` is decided in JS

| Call form | `this` is |
|---|---|
| `obj.m()` | `obj` |
| `const f = obj.m; f()` | `undefined` (strict) / global (sloppy) |
| `f.call(x)` / `f.apply(x)` | `x` |
| `f.bind(x)()` | `x` (permanently) |
| `new F()` | the new instance |
| arrow function | lexical `this` from enclosing scope (ignores all the above) |

#### Three common fixes

1. **`.bind(this)` in constructor:** `this.inc = this.inc.bind(this)`. Permanently bound; costs one function per instance.
2. **Arrow class field:** `inc = () => { this.n++ }`. Own property, lexical `this`. Can't be overridden via `super` in subclasses — that's the tradeoff.
3. **Wrap at call site:** `btn.addEventListener('click', () => c.inc())`. Keeps `inc` on the prototype; binding lives with the caller.

#### Why JS chose dynamic `this`

Not a mistake — a deliberate tradeoff that enables patterns Python can't express cleanly:

- **Function borrowing.** Any function can be called against any receiver, so methods from one type can operate on another:
  ```js
  Array.prototype.slice.call(arguments);               // turn arguments into a real array
  [].forEach.call(document.querySelectorAll('p'), …);  // iterate NodeList pre-ES6
  Object.prototype.toString.call(x);                   // reliable type tag: "[object Array]"
  ```
  Python would require `ClassName.method(obj, …)` and only works when `obj` is actually an instance of that class.

- **Callbacks and event handlers control `this` for you.** jQuery, DOM handlers, and `Array.prototype.forEach`'s `thisArg` all *set* `this` at call time — `this` inside a click handler is the element that fired the event:
  ```js
  $('button').on('click', function () { this.disabled = true; });
  ```
  Python callbacks can't receive an implicit receiver; you'd need a closure or explicit arg.

- **Mixins and composition without inheritance.** Copy a method onto any object and it just works — the object doesn't need to be an instance of a particular class:
  ```js
  Object.assign(target, { greet() { return `hi ${this.name}`; } });
  ```

- **`new` reuses the same function as a constructor.** Because `this` is decided at call site, `F()` and `new F()` can mean different things with the same function body.

- **Arrow functions opted *out* later.** ES6 added lexical `this` precisely for the callback case where you *don't* want dynamic binding. So JS now has both tools: dynamic `this` for flexibility, arrows for Python-like auto-binding.

**Mental model:**
- **Python:** `obj.method` = "give me a thing I can call later, `self` included."
- **JS:** `obj.method` = "give me the function; `this` is whatever the call site says."

The footgun (losing `this` in a callback) is the same property that enables borrowing, mixins, and `this`-as-receiver event handlers. Python's safety comes from giving up that flexibility.

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

## Best practices: instance vs class property

Rule of thumb: **pick the narrowest scope that fits the data's lifetime.**

| Data is... | Put it on... | How |
|---|---|---|
| Different per instance | Instance (own) | `x = 1` field or `this.x = 1` |
| Same for every instance, read-only behavior | Prototype | method in class body, or `static get` |
| Config/counter/registry for the class itself | Class (static) | `static x = 1` |
| Truly global | Module scope | `const X = 1` outside the class |

Guidelines:

- **Default to instance fields for state.** Reach for `static` only when the value is genuinely about the *class*, not any instance (e.g. `User.tableName`, `HttpClient.defaultTimeout`, an id counter).
- **Never put mutable objects on the prototype or as a `static` default that instances will mutate.** `Foo.prototype.items = []` or exposing `static items = []` and having instances push into it creates shared-state bugs. If it's mutable per-instance, make it an instance field.
- **Read statics via the class name, not `this`.** `User.tableName` is clearer than `this.constructor.tableName` and avoids surprises from subclass shadowing — unless subclass override is exactly what you want, in which case `this.constructor.x` is the right tool.
- **Don't reassign `static` fields to simulate globals.** If multiple unrelated callers mutate `Foo.count`, that's a module-level variable wearing a class costume. Move it out.
- **Prefer methods on the prototype (class body) over assigning functions in the constructor.** `this.greet = () => {...}` creates a new function per instance — fine for closures that capture `this`, wasteful otherwise.
- **Freeze statics that are meant as constants.** `static STATUSES = Object.freeze(['a','b'])` prevents accidental mutation.
- **Subclassing: remember statics are inherited by reference until shadowed.** `Sub.count` reads `Parent.count` until you write to `Sub.count`. Be explicit when you mean "each subclass gets its own counter" — initialize it on the subclass.

## Related

- [DOM: Node, Element, and Collections](./dom-node-element-collections.md) — concrete example of how DOM relies entirely on prototype methods.
