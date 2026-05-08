# `this` (JS) vs `self` (Python)

**TL;DR:** JS has two parallel systems — lexical scope for variable lookup, and a separate call-site mechanism for `this`. Python has one system — the receiver is always an explicit parameter (`self`/`cls`). JS trades predictability for flexibility: one function can serve multiple objects without classes. Arrow functions bridge the gap by opting `this` into the lexical system.

## Why `this` exists

A function needs to know which object it's operating on. Two design choices:

1. **Explicit parameter** (Python) — the receiver is a regular argument in the signature. Visible, predictable, no special rules.
2. **Implicit keyword** (JS) — the receiver is injected as `this` based on how the function is called. Hidden from the signature, governed by call-site rules.

Both solve the same problem (method sharing — one function, multiple objects). They differ in visibility and determination rules.

## `this` does not violate lexical scoping

Lexical scoping governs **variable name resolution** — "what does `x` refer to?" is answered by where the function is written in the source. `this` is not a variable lookup. It's a separate value injected into the execution context at call time. The two systems coexist without conflict:

| Question                             | System                             | Determined by                     |
| ------------------------------------ | ---------------------------------- | --------------------------------- |
| "What does the name `x` resolve to?" | Scope chain                        | Where the function is **written** |
| "What does `this` refer to?"         | `[[ThisValue]]` on the Function ER | How the function is **called**    |

## Why JS has more flexibility than Python

JS's call-site `this` enables patterns that Python's explicit `self` doesn't (or makes awkward):

### One function, many owners — without copying or classes

```js
function greet() {
  console.log(this.name);
}

const alice = { name: "Alice" };
const bob = { name: "Bob" };

alice.greet = greet;
bob.greet = greet;

alice.greet(); // "Alice"
bob.greet(); // "Bob"
```

Same function object, no duplication, no class. `this` adapts at call time. Python requires a class for `obj.method()` syntax, or you pass the object explicitly:

```python
def greet(obj):
    print(obj.name)

greet(alice)  # works, but no obj.greet() syntax without a class
```

### Borrowing methods between unrelated objects

```js
const arrayLike = { 0: "a", 1: "b", length: 2 };

Array.prototype.slice.call(arrayLike); // ["a", "b"]
```

`slice` was written for arrays, but `this` lets you point it at anything with the right shape. Python would require inheritance or explicit conversion.

### Dynamic dispatch without inheritance

```js
function serialize() {
  return JSON.stringify(this);
}

const config = { host: "localhost", port: 3000 };
const user = { name: "Alice", age: 30 };

serialize.call(config); // '{"host":"localhost","port":3000}'
serialize.call(user); // '{"name":"Alice","age":30}'
```

No shared base class, no interface. Python equivalent: `serialize(obj)` — works, but loses the method-call syntax.

### The tradeoff

|                   | JS (`this`)                                                | Python (`self`)                          |
| ----------------- | ---------------------------------------------------------- | ---------------------------------------- |
| Flexibility       | High — any function can operate on any object at call time | Lower — receiver fixed at method binding |
| Predictability    | Lower — depends on _how_ you call, not _what_ you wrote    | High — always visible in the signature   |
| Footguns          | Detachment, accidental `globalThis`, callback loss         | Almost none                              |
| Class requirement | None — ad-hoc method sharing works                         | Required for `obj.method()` syntax       |

### Why the difference exists

JS was designed (1995) for a world where objects are ad-hoc bags of properties and functions float between them freely. The `this` mechanism enables that without requiring a class system. Python was designed around classes from day one — `self` is always there because a class is always there.

Modern consensus: use arrow functions and explicit parameters when you don't need the flexibility. Use `this` when you genuinely want one function to serve multiple objects dynamically (prototype methods, framework hooks, mixins).

## How `this` / `self` is determined — by function kind

### JS — regular function

**Rule:** determined by the call site.

```js
function greet() {
  console.log(this.name);
}

const alice = { name: "Alice", greet };
const bob = { name: "Bob", greet };

alice.greet(); // "Alice"   — obj.fn() → this = obj
bob.greet(); // "Bob"     — obj.fn() → this = obj
greet(); // undefined — bare call → this = globalThis (or undefined in strict)

greet.call(alice); // "Alice"   — explicit via .call()
const bound = greet.bind(bob);
bound(); // "Bob"     — explicit via .bind()

new greet(); // this = fresh object (constructor call)
```

### JS — arrow function

**Rule:** lexical — inherits `this` from the enclosing scope at definition time. Call site is irrelevant. `.call()`/`.bind()`/`new` cannot override it.

```js
function outer() {
  const arrow = () => console.log(this.name);
  arrow(); // uses outer's this
  arrow.call({ name: "ignored" }); // still uses outer's this
}

outer.call({ name: "Alice" }); // "Alice" — arrow captured outer's this
```

### JS — class method

**Rule:** same as regular function (it _is_ a regular function on the prototype). Loses binding when detached.

```js
class Dog {
  name = "Rex";
  bark() {
    console.log(this.name);
  }
}

const d = new Dog();
d.bark(); // "Rex" — obj.method() → this = obj

const fn = d.bark;
fn(); // undefined — detached, bare call, this lost
```

### JS — class field arrow

**Rule:** lexical `this`, bound to the instance at construction time. Never loses binding.

```js
class Dog {
  name = "Rex";
  bark = () => {
    console.log(this.name);
  };
}

const d = new Dog();
const fn = d.bark;
fn(); // "Rex" — still bound, arrow captured this at construction
setTimeout(d.bark, 0); // "Rex" — safe to pass around
```

### Python — regular method

**Rule:** always explicit. `obj.method()` desugars to `Class.method(obj)`.

```python
class Dog:
    def __init__(self, name):
        self.name = name

    def bark(self):
        print(self.name)

d = Dog("Rex")
d.bark()          # "Rex" — Python passes d as self
fn = d.bark       # creates a bound method — self is already baked in
fn()              # "Rex" — no detachment problem
```

### Python — `@staticmethod`

**Rule:** no `self`/`cls` passed. Plain function namespaced under the class.

```python
class MathUtils:
    @staticmethod
    def add(a, b):
        return a + b

MathUtils.add(1, 2)  # 3 — no receiver at all
```

### Python — `@classmethod`

**Rule:** first arg is the _class_ (`cls`), not the instance. Determined by the class the method is looked up on.

```python
class Animal:
    species = "unknown"

    @classmethod
    def describe(cls):
        print(cls.species)

class Dog(Animal):
    species = "canine"

Dog.describe()     # "canine" — cls = Dog
Animal.describe()  # "unknown" — cls = Animal
```

### Python — lambda

**Rule:** same as regular function. `self` must be an explicit parameter if needed. No implicit receiver.

```python
class Foo:
    def get_printer(self):
        return lambda: print(self.x)  # self captured via closure, not magic

f = Foo()
f.x = 42
printer = f.get_printer()
printer()  # 42 — self was closed over explicitly
```

## The JS detachment problem (Python doesn't have it)

```js
class Timer {
  seconds = 0;
  tick() {
    this.seconds++;
  }
}

const t = new Timer();
setInterval(t.tick, 1000); // BUG: this = undefined (strict) or globalThis
```

Extracting `t.tick` detaches it from `t`. The call site becomes `tick()` (bare call), so `this` is no longer `t`.

Fixes:

- `setInterval(() => t.tick(), 1000)` — arrow wraps the call, preserving `t.method()` syntax
- `tick = () => { ... }` — class field arrow, lexical `this`
- `setInterval(t.tick.bind(t), 1000)` — explicit bind

Python doesn't have this problem because `obj.method` creates a **bound method object** — `self` is already baked in at attribute access time, not at call time.

## JS arrow functions vs Python methods — similar but different

Both give you a "safe to pass around" function with a fixed receiver. But the mechanism differs:

|                             | JS arrow function                                                | Python bound method                               |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------- |
| Receiver comes from         | Enclosing scope's `this` (lexical capture)                       | The specific instance accessed on (`obj.method`)  |
| What it captures            | Whatever `this` happened to be — might not be an instance at all | Always the instance (`self`)                      |
| Can serve multiple objects? | No — locked to the captured `this`                               | No — locked to the instance                       |
| Mechanism                   | Closure over `this`                                              | Descriptor protocol creates a bound method object |

**Same practical effect — different mechanism:**

```js
// JS: arrow captures enclosing this — not necessarily an instance
function factory() {
  return () => console.log(this.name);
}

const arrow = factory.call({ name: "Alice" }); // this = ad-hoc object
arrow(); // "Alice" — captured from factory's this, no class involved
```

```python
# Python: bound method is always tied to a specific instance
class Dog:
    def __init__(self, name):
        self.name = name
    def bark(self):
        print(self.name)

d = Dog("Rex")
fn = d.bark   # bound method — self = d, always
fn()          # "Rex"

# Closest Python equivalent to the JS factory pattern is a closure:
def factory(obj):
    return lambda: print(obj.name)

printer = factory(type("X", (), {"name": "Alice"})())
printer()  # "Alice" — closure over a variable, not a bound method
```

The difference: JS arrows capture `this` from _whatever scope they're inside_ — a class constructor, a plain function called with `.call()`, or a global script. Python has no floating `self` that gets captured — you either have a bound method (always an instance) or you explicitly close over a variable.

## Key takeaway

Python: one system, explicit, no surprises. JS: two systems (scope chain + `this`), with more flexibility (ad-hoc method sharing, method borrowing, dynamic dispatch without classes) at the cost of predictability. Arrow functions are the escape hatch that unifies them by making `this` lexical.
