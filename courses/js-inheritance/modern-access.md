# Modern Prototype Access

> **TL;DR:** `Object.getPrototypeOf` and `Object.setPrototypeOf` are the official APIs for reading/writing the `[[Prototype]]` internal slot (the hidden link that drives the chain walk). They bypass property lookup entirely — no chain walk, no accessor magic, no shadowing risk. Prefer wiring `[[Prototype]]` at creation time with `Object.create(proto)` over mutating it after the fact with `setPrototypeOf`, because post-creation mutation forces the engine to throw away optimized hidden classes and deoptimize.

## Why `__proto__` isn't enough

Every `__proto__` failure mode comes from it being a **property** that depends on the chain walk reaching the accessor on `Object.prototype`. Cut the chain, shadow the accessor, or feed `"__proto__"` as a string key from user input, and it breaks. See [`__proto__`](dunder-proto.md) for the full breakdown.

The modern APIs sidestep all of this by going straight to the engine's internal slot.

## `Object.getPrototypeOf(obj)` — ES5

The first official way to read `[[Prototype]]`. It's a function call, not a property lookup.

```js
const parent = { role: "parent" };
const child = Object.create(parent);

Object.getPrototypeOf(child) === parent; // true
```

Works where `__proto__` doesn't:

```js
const bare = Object.create(null);
bare.__proto__; // undefined — no accessor in chain
Object.getPrototypeOf(bare); // null — correct answer
```

**Primitives:** ES6 changed the behavior from throwing to auto-boxing — mirrors the auto-boxing from the [deep dive](prototype-deep-dive.md):

```js
Object.getPrototypeOf(42); // Number.prototype
Object.getPrototypeOf("hello"); // String.prototype
Object.getPrototypeOf(true); // Boolean.prototype
```

## `Object.setPrototypeOf(obj, proto)` — ES6

The official way to write `[[Prototype]]`. Same advantages: function call, no property lookup, works on `Object.create(null)` objects.

```js
const bare = Object.create(null);
Object.setPrototypeOf(bare, {
  greet() {
    return "hi";
  },
});
bare.greet(); // "hi" — works fine, no accessor needed
```

### The performance warning

Mutating `[[Prototype]]` after creation is slow. The engine builds optimized internal structures (hidden classes / shapes) based on the prototype chain at creation time. Changing `[[Prototype]]` after the fact:

1. Throws away the cached shape for that object.
2. Deoptimizes any compiled code that assumed the old chain.
3. Rebuilds from scratch.

This isn't "slightly slower" — it can tank performance for the entire program if it happens in a hot path, because deoptimization cascades.

**The rule:** Wire `[[Prototype]]` at creation time. Use `Object.create(proto)` or constructor functions / classes. Reach for `setPrototypeOf` only when you have an existing object and _must_ change its chain (polyfills, framework internals). Application code should almost never need it.

## `Object.create` vs `Object.setPrototypeOf`

Both can wire a prototype, but they're for different moments:

```js
// Wire at birth — preferred
const child = Object.create(parent);

// Rewire after birth — works, but slower
const obj = {};
Object.setPrototypeOf(obj, parent);
```

`Object.create` is the default choice. The chain is right from the start — no mutation, no deoptimization. `setPrototypeOf` exists for the rare case where rewiring is unavoidable.

## `Reflect` variants — ES6

`Reflect.getPrototypeOf` and `Reflect.setPrototypeOf` behave almost identically but differ in error handling:

```js
// Object version — coerces non-objects (auto-boxing), throws on null/undefined
Object.getPrototypeOf(42); // Number.prototype

// Reflect version — throws TypeError on any non-object
Reflect.getPrototypeOf(42); // TypeError
```

`Reflect.setPrototypeOf` returns a boolean instead of throwing on failure (e.g., non-extensible object). Useful in Proxy traps where you need to signal success/failure. For everyday code, `Object` versions are standard.

## Reliability comparison

| Problem                                   | `__proto__`                           | `getPrototypeOf` / `setPrototypeOf` |
| ----------------------------------------- | ------------------------------------- | ----------------------------------- |
| `Object.create(null)`                     | Dead — no accessor in chain           | Works — reads/writes slot directly  |
| Shadowed by `defineProperty`              | Dead — own property wins              | Works — ignores properties entirely |
| Custom accessor shadows it                | Runs your code, not the engine's      | Works — ignores properties entirely |
| `__proto__` as string key from user input | Triggers setter → prototype pollution | N/A — not a property name           |
| Non-browser environments                  | Annex B — technically optional        | Main spec — required everywhere     |

## API summary

| API                                  | Added | Does what                               | Notes                                         |
| ------------------------------------ | ----- | --------------------------------------- | --------------------------------------------- |
| `Object.getPrototypeOf(obj)`         | ES5   | Reads `[[Prototype]]`                   | Always reliable, auto-boxes primitives (ES6+) |
| `Object.setPrototypeOf(obj, proto)`  | ES6   | Writes `[[Prototype]]`                  | Works but avoid — prefer wiring at creation   |
| `Object.create(proto)`               | ES5   | Creates object with `[[Prototype]]` set | The right way to wire chains                  |
| `Reflect.getPrototypeOf(obj)`        | ES6   | Reads `[[Prototype]]`                   | Strict — throws on non-objects                |
| `Reflect.setPrototypeOf(obj, proto)` | ES6   | Writes `[[Prototype]]`                  | Returns boolean instead of throwing           |
