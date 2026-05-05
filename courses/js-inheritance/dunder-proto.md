# `__proto__`

> **TL;DR:** `__proto__` is not a normal property — it's an **accessor** (getter/setter) on `Object.prototype` that reads and writes the `[[Prototype]]` internal slot. Its magic depends entirely on the chain walk reaching that accessor. Cut the chain (`Object.create(null)`) or shadow it (`Object.defineProperty`), and `__proto__` becomes a plain string key with no special powers. It's standardized in ES6 Annex B (required for browsers, deprecated for new code).

## What it actually is

`__proto__` was invented by Mozilla's SpiderMonkey engine as a non-standard way to access `[[Prototype]]` before any official API existed. Other browsers copied it for web compatibility.

It's an **accessor property** — a getter/setter pair — defined on `Object.prototype`:

```js
Object.getOwnPropertyDescriptor(Object.prototype, "__proto__");
// {
//   get: [Function: get __proto__],
//   set: [Function: set __proto__],
//   enumerable: false,
//   configurable: true
// }
```

When you write `obj.__proto__`:

1. Engine looks for `__proto__` on `obj` — not found.
2. Chain walk hits `Object.prototype`, finds the accessor.
3. The **getter** runs, returns the `[[Prototype]]` internal slot.

When you write `obj.__proto__ = X`, the **setter** runs and rewires `[[Prototype]]`.

The magic is in the accessor, not the name.

## The name

"Dunder proto" — "dunder" is short for "double underscore." The `__name__` convention was borrowed from Python, where it signals "internal/magic." In JS, `__proto__` is the only widely-used dunder.

## ES6 and Annex B

ES6 (ES2015) couldn't ignore `__proto__` (too much web code depended on it) but didn't want to bless it either. The compromise: **Annex B** — a spec section titled "Additional ECMAScript Features for Web Browsers."

Annex B means:

- Browsers **must** implement it (web compatibility).
- Non-browser environments (Node) **may** skip it (in theory — none actually do).
- You **shouldn't** use it in new code.

So `__proto__` is simultaneously standardized, deprecated, and universal.

## When the magic disappears

The accessor only fires if the chain walk reaches `Object.prototype`. Three scenarios break that:

### `Object.create(null)` — no chain to walk

```js
const bare = Object.create(null);
bare.__proto__ = {
  greet() {
    return "hi";
  },
};

bare.greet; // undefined — no prototype was wired
bare.__proto__; // { greet: [Function] } — plain data property
Object.getPrototypeOf(bare); // null — [[Prototype]] untouched
Object.keys(bare); // ["__proto__"] — normal enumerable key
```

No chain to `Object.prototype` → no accessor found → assignment creates a plain property named `"__proto__"`. This is why `Object.create(null)` objects are safe dictionaries — user-supplied `"__proto__"` keys can't accidentally rewire the prototype.

### `Object.defineProperty` — shadow the accessor

`Object.defineProperty` bypasses the accessor entirely. It doesn't do a property lookup — it directly manipulates the property descriptor on the target object. The inherited setter on `Object.prototype` never fires. The prototype chain keeps working exactly as before — `[[Prototype]]` still points to `Object.prototype`, so inherited methods like `toString` resolve normally.

```js
const obj = {};
Object.defineProperty(obj, "__proto__", { value: { y: 2 } });

obj.__proto__; // { y: 2 } — own data property
Object.getPrototypeOf(obj); // Object.prototype — [[Prototype]] untouched
obj.toString; // [Function] — chain still works
```

`obj` now has both:

- An own data property named `"__proto__"` holding `{ y: 2 }`.
- Its `[[Prototype]]` still pointing to `Object.prototype`, completely untouched.

The own property shadows the inherited accessor permanently. Even `obj.__proto__ = something` won't rewire the prototype anymore — it hits the own data property, not the setter. (And since `defineProperty` defaults `writable` to `false`, the assignment silently fails in sloppy mode.)

### Own accessor via `defineProperty` — your getter, not the engine's

```js
const o2 = {};
Object.defineProperty(o2, "__proto__", {
  get() {
    return { x: 2 };
  },
});

o2.__proto__; // { x: 2 } — your getter
o2.x; // undefined — [[Prototype]] was never rewired
Object.getPrototypeOf(o2); // Object.prototype — real chain link
```

Your accessor shadows the inherited one. Reading `o2.__proto__` calls your getter — but the engine doesn't use `__proto__` for property lookup. It reads the `[[Prototype]]` slot directly. Your getter has no influence on the chain walk.

## Behavior summary

| Scenario                               | `obj.__proto__ = X` does what?                                 |
| -------------------------------------- | -------------------------------------------------------------- |
| Normal object (`{}`)                   | Setter on `Object.prototype` fires → rewires `[[Prototype]]`   |
| `Object.create(null)`                  | No accessor in chain → plain data property named `"__proto__"` |
| Own accessor via `defineProperty`      | Your accessor runs → whatever you coded                        |
| Own data property via `defineProperty` | Data property shadows accessor → no `[[Prototype]]` change     |

## `__proto__` vs `Object.getPrototypeOf`

`obj.__proto__` and `Object.getPrototypeOf(obj)` can return different things when the accessor is shadowed:

```js
const o1 = {};
Object.defineProperty(o1, "__proto__", { value: { y: 2 } });

o1.__proto__; // { y: 2 } — the own data property
Object.getPrototypeOf(o1); // Object.prototype — the real [[Prototype]]
```

`Object.getPrototypeOf` always reads the internal slot directly — it doesn't go through property lookup at all. It's the only reliable way to inspect `[[Prototype]]`.

## Prototype pollution

Regular objects are vulnerable to prototype pollution via `__proto__` keys from untrusted input:

```js
const regular = {};
regular["__proto__"] = { x: 1 }; // triggers the setter — rewires [[Prototype]]

const safe = Object.create(null);
safe["__proto__"] = { x: 1 }; // just a data property, no magic
```

If you're building lookup maps from user input (query params, JSON keys, config), `Object.create(null)` or `Map` sidesteps the risk.
