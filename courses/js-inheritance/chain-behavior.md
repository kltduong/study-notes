# Prototype Chain & Behavior

> **TL;DR:** Property access follows two rules: **reads walk up** the `[[Prototype]]` chain, **writes stay put** on the target object. This asymmetry creates shadowing (own properties hiding inherited ones), keeps per-instance state safe, and explains why in-place mutation of inherited references is a trap. Every chain terminates at `Object.prototype` → `null`. Enumerability controls which properties show up during iteration, not whether they're accessible. Mutating `[[Prototype]]` at runtime kills engine optimizations — wire prototypes at creation time.

## Shadowing — reads walk up, writes stay put

When you assign a property to an object, the assignment **always** creates or updates an own property on that object. It never walks the chain. Reads do the opposite — they walk the chain until the property is found or `null` is reached.

```js
const parent = { x: 10 };
const child = Object.create(parent);

child.x; // 10 — read walks up, finds parent.x
child.x = 99; // write stays put — creates own property on child

child.x; // 99 — own property found first, chain walk stops
parent.x; // 10 — untouched
```

The own property on `child` now **shadows** `parent.x`. The inherited value isn't deleted — it's just unreachable from `child`. Remove the shadow and it reappears:

```js
delete child.x;
child.x; // 10 — parent.x visible again
```

### Compound operators are secretly read-then-write

`this.count++` desugars to `this.count = this.count + 1`. Two operations, two different rules:

1. **Read** `this.count` (right side) — chain walk finds the value on the prototype.
2. **Write** `this.count = ...` (left side) — assignment lands on `this` (the calling object). Own property created.

```js
const parent = {
  count: 0,
  increment() {
    this.count++;
  },
};

const child = Object.create(parent);
child.increment();

child.count; // 1 — own property, just created by the write
parent.count; // 0 — the read found it, but the write never touched it
```

This applies to all compound operators: `++`, `--`, `+=`, `-=`, etc. Decompose them into read + write and the two rules explain everything.

This is _by design_ — it means each object gets its own instance state automatically. If writes walked the chain, every object sharing a prototype would fight over the same properties.

### The mutable reference trap

The read-up/write-down rule only protects you when there's an **assignment**. In-place mutation of a reference type bypasses shadowing entirely:

```js
const parent = { tags: ["shared"] };
const child1 = Object.create(parent);
const child2 = Object.create(parent);

child1.tags.push("oops");
// child1.tags is a READ — walks chain, finds parent's array
// .push() mutates that array in place — no assignment, no shadowing

child1.tags; // ["shared", "oops"]
child2.tags; // ["shared", "oops"] — same array, both see the mutation
parent.tags; // ["shared", "oops"] — the prototype's array was mutated
```

Compare with spread + assignment, which triggers shadowing:

```js
child1.tags = [...child1.tags, 1];
// READ child1.tags (chain walk) → spread into new array → WRITE to child1
// Assignment triggers shadowing — child1 gets its own array

child1.tags; // ["shared", "oops", 1] — own copy
parent.tags; // ["shared", "oops"] — untouched by the assignment
```

The principle: **mutation without assignment bypasses shadowing.** Any method that modifies in place (`.push()`, `.splice()`, setting nested properties like `obj.nested.x = ...`) is a trap when the value lives on the prototype. Assignment (`=`) is what triggers the "write stays put" rule.

Fix: assign mutable values as own properties during initialization (in a constructor or right after `Object.create`), not on the prototype.

## Chain termination at `null`

Every prototype chain ends at `Object.prototype`, whose own `[[Prototype]]` is `null`:

```js
Object.getPrototypeOf(Object.prototype); // null
```

When a lookup reaches `null`, the engine stops and returns `undefined`. Trying to call that `undefined` as a function gives `TypeError`:

```js
const obj = {};
obj.nope; // undefined — chain walked to null, never found
obj.nope(); // TypeError: obj.nope is not a function
```

Cycles are forbidden — `Object.setPrototypeOf` throws `TypeError` if you try to create a circular chain. The chain is always finite.

## Enumerability and iteration

Every property has a **descriptor** with `value`, `writable`, `enumerable`, and `configurable` flags. Properties created normally (assignment or literal) default to `enumerable: true`. Built-in methods on `Object.prototype` are `enumerable: false` — that's why `for...in` walks the whole chain but never yields `toString` or `hasOwnProperty`.

```js
Object.getOwnPropertyDescriptor(Object.prototype, "toString");
// { value: ƒ, writable: true, enumerable: false, configurable: true }
```

You can create non-enumerable own properties:

```js
const obj = {};
Object.defineProperty(obj, "hidden", { value: 42, enumerable: false });

obj.hidden; // 42 — normal access finds it
Object.keys(obj); // [] — enumerable filter hides it
```

Enumerability controls **iteration visibility**, not access. The property is still there and reachable by name.

### Iteration tools compared

| Tool                             | Scope        | Enumerable only? | Includes symbols?  |
| -------------------------------- | ------------ | ---------------- | ------------------ |
| `Object.keys()`                  | Own          | Yes              | No                 |
| `Object.getOwnPropertyNames()`   | Own          | No (all)         | No                 |
| `Object.getOwnPropertySymbols()` | Own          | No (all)         | Yes (symbols only) |
| `for...in`                       | Entire chain | Yes              | No                 |
| `Reflect.ownKeys()`              | Own          | No (all)         | Yes                |

`Reflect.ownKeys()` is the "show me everything on this one object" tool — strings + symbols, enumerable or not. Own only, no chain walk.

## Don't mutate prototypes at runtime

`Object.setPrototypeOf()` rewires an object's `[[Prototype]]` after creation. It works, but engines strongly discourage it.

V8 (and other engines) optimize property access using **hidden classes** (also called "shapes" or "maps"). When an object is created, the engine assigns a hidden class based on its structure and prototype. Property lookups become fast pointer offsets instead of chain walks.

`setPrototypeOf()` **invalidates** that hidden class:

1. Deoptimizes the object — throws away compiled fast paths.
2. Can deoptimize _other_ objects that shared the same hidden class.
3. Forces the engine to rebuild optimization data from scratch.

This isn't a one-time cost — it can poison the optimization pipeline for the program's lifetime.

The rule: **wire prototypes at creation time** (`Object.create`, `new`, `class extends`), not after the fact. If you're reaching for `setPrototypeOf` on an existing object, it's almost always a design smell. The one legitimate use case is polyfilling or subclassing builtins in older environments — and even then, `class extends` is preferred.
