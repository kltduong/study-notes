# Class & `this` — Draft

## Plan (teaching order)

- [x] Sub-part 1: Class methods — non-constructable, `this` follows the same Reference-base rule
- [ ] Sub-part 2: Method extraction and `this`-loss — the class-specific shape of the problem
- [ ] Sub-part 3: Field initializers — when they run, what `this` they see, arrow-field pattern
- [ ] Sub-part 4: `super()` as `this`-provider in derived constructors — `this`-TDZ, the uninitialized state
- [ ] Sub-part 5: Synthesis — worked example tying the pieces together

---

## Teaser

```js
"use strict";                                         // L1
class Timer {                                         // L2
  count = 0;                                          // L3
  start() {                                           // L4
    setInterval(function () {                         // L5
      this.count++;                                   // L6
      console.log(this.count);                        // L7
    }, 1000);                                         // L8
  }                                                   // L9
  startArrow() {                                      // L10
    setInterval(() => {                               // L11
      this.count++;                                   // L12
      console.log(this.count);                        // L13
    }, 1000);                                         // L14
  }                                                   // L15
}                                                     // L16

const t = new Timer();                                // L17
t.start();                                            // L18
t.startArrow();                                       // L19
```

- **L18 (`start`):** `setInterval` invokes the callback as a plain call → Reference base = `undefined` (strict) → `this.count++` throws **TypeError**.
- **L19 (`startArrow`):** Arrow has no `this` machinery → `[[OuterEnv]]` chain walk → `startArrow`'s ER → `[[ThisValue]] = t` → `this.count++` mutates `t.count` → 1, 2, 3, …

---

## Sub-part 1: Class methods — non-constructable, same `this` rule

### Methods are ordinary functions with two structural differences

A class method (defined via method shorthand inside a `class` body) is an ordinary function object — same `[[Call]]`, same `this`-determination pipeline. But it differs from a function declaration/expression in two ways:

| Property | Function declaration/expression | Class method (shorthand) |
|---|---|---|
| `[[IsConstructor]]` | `true` | `false` |
| `[[HomeObject]]` | not set | set to the class's `.prototype` object |

**`[[IsConstructor]] = false`** — you cannot `new` a method. This is the same gate arrows hit, but for a different reason: arrows lack `this` machinery entirely; methods *have* `this` machinery but are semantically "not standalone constructors." The engine enforces this at creation time based on syntactic form.

**`[[HomeObject]]`** — enables `super.method()` calls. It's a static link to the prototype object the method was defined on. Not relevant to `this` determination, but it's the other structural difference worth noting (it feeds into js-inheritance).

### `this` follows the exact same Reference-base rule

```js
"use strict";                                         // L1
class Dog {                                           // L2
  name;                                               // L3
  constructor(n) { this.name = n; }                   // L4
  bark() { return this.name + " barks"; }             // L5
}                                                     // L6

const d = new Dog("Rex");                             // L7
d.bark();                                             // L8 — Reference: base=d, name="bark" → this=d → "Rex barks"

const fn = d.bark;                                    // L9 — GetValue extracts the function, Reference discarded
fn();                                                 // L10 — Reference: base=ER → this=undefined → TypeError
```

No class-specific `this` rule. L8 works because `d.bark()` is a member-expression call → Reference base = `d`. L10 fails because `fn()` is a plain identifier call → Reference base = ER → `undefined` in strict.

> **Aside —** Classes are always strict. The class body is implicitly strict regardless of surrounding code. So the sloppy-mode `globalThis` fallback never applies inside a class method — `this` is `undefined` on plain calls, never the global object.

### Why non-constructable matters

```js
new d.bark();                                         // TypeError: d.bark is not a constructor
```

The engine checks `[[IsConstructor]]` before running `[[Construct]]`. Method shorthand → `false` → immediate TypeError. This is a **design-time guarantee**: methods can't accidentally be used as constructors. No `.prototype` property is created for them either (saves memory, signals intent).
