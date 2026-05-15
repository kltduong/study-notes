# Building Prototype Chains — Draft

## Plan (teaching order)

- [x] 1. Frame: multi-level inheritance, four wiring mechanisms (same outcome, different ergonomics)
- [x] 2. Teaser — pre-2011 `Dog.prototype = new Animal()`: what's wrong?
- [x] 3. Pre-2011 wiring + the two problems it reveals (resource waste, `.constructor` confusion)
- [x] 4. `Object.create(Animal.prototype)` — ES5 fix, what changes vs pre-2011
- [ ] 5. `Object.setPrototypeOf` — ES6, why rarely used despite being symmetrical
- [ ] 6. `call()` — running parent constructor inside child for instance props
- [ ] 7. `class extends` — what it desugars to; `super()` requirement
- [ ] 8. 3-level chain worked example: constructor form ↔ class form
- [ ] 9. Unified comparison table + the duplication problem fully named
- [ ] 10. End-of-chunk understanding check

---

## 1. Frame — what "building a chain" means

You've wired a single-level chain many times:

```js
function Dog(breed) {
  this.breed = breed;
}
Dog.prototype.bark = function () {
  return 'woof';
};

const rex = new Dog('Shepherd');

// rex ──▶ Dog.prototype ──▶ Object.prototype ──▶ null

// rex.[[Prototype]]             → Dog.prototype
// Dog.prototype.[[Prototype]]   → Object.prototype
// Object.prototype.[[Prototype]]→ null
```

A **multi-level chain** just inserts another link in the middle:

```js
// rex ──▶ Dog.prototype ──▶ Animal.prototype ──▶ Object.prototype ──▶ null
```

The whole chunk is about **how you make that middle arrow exist**. Four mechanisms, same chain:

| Era                  | Mechanism                                              |
| -------------------- | ------------------------------------------------------ |
| Pre-2011             | `Dog.prototype = new Animal()`                         |
| ES5 (2009)           | `Dog.prototype = Object.create(Animal.prototype)`      |
| ES6 (2015)           | `Object.setPrototypeOf(Dog.prototype, Animal.prototype)` |
| ES6 (2015)           | `class Dog extends Animal { … }`                       |

Each later mechanism exists because the earlier one had problems. The two pre-2011 problems are the motivation for everything that follows — so we name them precisely first.

---

## 2 & 3. The pre-2011 approach and its two problems

Recap of the teaser:

```js
function Animal(name) { this.name = name; }
Animal.prototype.eat = function () { return `${this.name} eats`; };

function Dog(name, breed) {
  this.name = name;
  this.breed = breed;
}
Dog.prototype = new Animal();        // ◀── the wiring step
Dog.prototype.bark = function () { return `${this.name} barks`; };

const rex = new Dog('Rex', 'labrador');
rex.eat();                // 'Rex eats'   ✓ chain works
rex.constructor.name;     // 'Animal'     ✗ lies

// rex ──▶ Dog.prototype ──▶ Animal.prototype ──▶ Object.prototype ──▶ null

// rex.[[Prototype]]             → Dog.prototype (which IS an Animal instance)
// Dog.prototype.[[Prototype]]   → Animal.prototype
// Animal.prototype.[[Prototype]]→ Object.prototype
// Object.prototype.[[Prototype]]→ null
```

The wiring _functionally works_ — `eat` is reachable, the chain is correct. But two things are wrong with **how** we built it.

### Problem A — Dog.prototype is an Animal instance, not a clean prototype

`new Animal()` does what `new` always does:

1. Creates a fresh object with `[[Prototype]] = Animal.prototype`. ✓ (this is the part we wanted)
2. **Runs the `Animal` constructor with `this` bound to that object.** ✗ (this is the part we didn't want)

So `Dog.prototype` ends up with `name: undefined` as an **own** property (from `this.name = name` with `name` being `undefined`). It also paid the cost of running the constructor for nothing.

The deeper problem is conceptual: `Dog.prototype` is supposed to be a _container of shared behavior_ for dogs. Instead, it's an _animal_. If Animal's constructor did real work (allocated buffers, registered event listeners, mutated globals) we'd be running that work once at class-definition time, with garbage arguments, into a prototype object. Wrong layer.

### Problem B — `.constructor` lies

Every function `F` gets a `.prototype` whose `.constructor` points back to `F`. That's where `instance.constructor` resolves to:

```
instance ─▶ F.prototype { constructor: F, … }
```

When we **overwrite** `Dog.prototype` (rather than mutate it), we throw away the object that had `constructor: Dog`. The chain walk for `.constructor` now skips past Dog.prototype (no own constructor there — it's an Animal instance with only `name` and `bark`) and lands at `Animal.prototype.constructor`, which is `Animal`.

So `rex.constructor === Animal`. Code that uses `instance.constructor` for introspection (or `instance instanceof rex.constructor`) is now broken. People worked around this by manually re-pinning:

```js
Dog.prototype = new Animal();
Dog.prototype.constructor = Dog;   // patch the lie
```

…which is exactly the kind of fragile boilerplate the ES5 fix removes.

### Root cause (unified view)

Both problems share one origin: **pre-2011 JS had no API for "just give me an object whose `[[Prototype]]` is X."** `new Constructor()` was the closest, but it bundles three things:

1. Allocate a new object linked to `Constructor.prototype`. ◀── all we wanted
2. Run `Constructor` with `this` bound to that object. ◀── caused Problem A
3. Return that new object (forcing us to **assign** it, overwriting Dog.prototype). ◀── caused Problem B

The ES5 fix isolates step 1.

---

## 4. `Object.create(Animal.prototype)` — the ES5 fix

`Object.create(proto)` does **exactly one thing**: allocate a fresh empty object whose `[[Prototype]]` is `proto`. No constructor runs. No instance state.

```js
function Animal(name) { this.name = name; }
Animal.prototype.eat = function () { return `${this.name} eats`; };

function Dog(name, breed) {
  this.name = name;
  this.breed = breed;
}

Dog.prototype = Object.create(Animal.prototype);   // ◀── pure link, nothing else
Dog.prototype.constructor = Dog;                   // ◀── re-pin (we still overwrote)
Dog.prototype.bark = function () { return `${this.name} barks`; };

const rex = new Dog('Rex', 'labrador');
rex.eat();                // 'Rex eats'    ✓
rex.constructor.name;     // 'Dog'         ✓ once we re-pin
```

### What changed vs pre-2011

| Issue                          | `new Animal()`                              | `Object.create(Animal.prototype)`            |
| ------------------------------ | ------------------------------------------- | -------------------------------------------- |
| Runs Animal constructor?       | Yes — pollutes Dog.prototype                | **No**                                       |
| `name: undefined` own prop?    | Yes (from `this.name = name`)               | **No** — Dog.prototype starts empty          |
| Dog.prototype's `[[Prototype]]`| `Animal.prototype` (via the `new` step 1)   | **`Animal.prototype`** (directly, no detour) |
| Need to re-pin `.constructor`? | Yes (still overwritten)                     | Yes (still overwritten)                      |

Problem A is gone. Problem B is **untouched** — we still overwrite `Dog.prototype`, so `.constructor` still needs re-pinning manually. The root cause is the same: `Dog.prototype = …` discards the original object (and its `constructor: Dog` property). To eliminate Problem B, we'd need to keep the original `Dog.prototype` object and only change _its_ `[[Prototype]]`. That's the niche `setPrototypeOf` fills (next).

### The shape of `Object.create` (preview)

`Object.create` is more than a chain-wiring tool — it's the general "make an object with this prototype" primitive. You've seen it as part of [instantiation patterns](instantiation.md) (the Prototypal pattern). Here we're applying the same primitive at a different layer: not to create _instances_, but to create the _prototype object_ that future instances will link to.

> **Aside —** `Object.create` also accepts a second argument: a property-descriptors map (`{ name: { value: 'Rex', writable: true, … } }`). Rarely used inline for chain-building. Mentioned for completeness.

