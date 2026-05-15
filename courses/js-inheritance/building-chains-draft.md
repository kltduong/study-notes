# Building Prototype Chains — Draft

## Plan (teaching order)

- [x] 1. Frame: multi-level inheritance, four wiring mechanisms (same outcome, different ergonomics)
- [x] 2. Teaser — pre-2011 `Dog.prototype = new Animal()`: what's wrong?
- [x] 3. Pre-2011 wiring + the two problems it reveals (resource waste, `.constructor` confusion)
- [ ] 4. `Object.create(Animal.prototype)` — ES5 fix, what changes vs pre-2011
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
