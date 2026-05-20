# 1. Core Iteration Abstractions — Draft

## 1.1. Plan (teaching order)

- [x] Three structural shapes — the taxonomy (filter: subset, map: 1-to-1, reduce: many-to-one)
- [ ] `map` deep dive — structure preservation, the functor intuition
- [x] `map` deep dive — structure preservation, the functor intuition
- [ ] `filter` deep dive — predicate as selector, Boolean coercion trick
- [ ] `reduce` intro — the accumulator loop, why it's the most general
- [ ] Composition of the three — chaining, data-flow pipelines, when to fuse
- [ ] Relationship to `for` loops — what you gain, what you lose

---

## 1.2. Three structural shapes — the taxonomy

Every array iteration method in JS falls into one of three structural shapes. The shape tells you the **relationship between input and output** — before you know anything about the callback logic.

| Shape | Method | Input → Output | Size relationship | Per-element dependency |
|-------|--------|---------------|-------------------|----------------------|
| **Subset** | `filter` | `T[]` → `T[]` | `output.length ≤ input.length` | Each element independently tested; kept or discarded |
| **1-to-1 transform** | `map` | `T[]` → `U[]` | `output.length === input.length` | Each output element derived from exactly one input element |
| **Aggregation** | `reduce` | `T[]` → `U` | Collection → single value | All elements contribute to one accumulated result |

### 1.2.1. Why this taxonomy matters

These aren't just three methods — they're three **operations on collections** that show up in every language with first-class functions. Python has `filter()`, `map()`, `functools.reduce()`. Haskell, Rust, Ruby, Clojure — same trio, same shapes.

The taxonomy tells you which tool to reach for based on what you need:

- Need the same number of items, each transformed? → `map`
- Need fewer items, unchanged? → `filter`
- Need to collapse a collection into something else entirely? → `reduce`

If you find yourself writing a `for` loop, ask: which shape is this? If it's one of the three, the named method communicates intent better than the loop body.

### 1.2.2. The teaser revisited

```js
const words = ["hello", "world", "foo", ""];  // L1

const result = words
  .filter(Boolean)                            // L2 — subset: 4 items → 3 (discard "")
  .map(w => w[0].toUpperCase())               // L3 — 1-to-1: 3 strings → 3 chars
  .reduce((acc, ch) => acc + ch, "");          // L4 — aggregation: 3 chars → 1 string

console.log(result);                          // L5 — "HWF"
```

Each step has a clear structural role. The chain reads as a pipeline: select → transform → aggregate. This is the fundamental composition pattern for data processing in FP style.



---

## 1.3. `map` — structure-preserving transformation

### 1.3.1. The contract

`map` makes a strong promise: **the output array has the same length as the input, and element `i` of the output is derived solely from element `i` of the input.**

```js
const nums = [1, 2, 3, 4];           // L1
const doubled = nums.map(x => x * 2); // L2 — [2, 4, 6, 8]
```

The callback `x => x * 2` sees one element at a time. It has no access to the accumulator, no access to "the result so far." Each call is independent.

### 1.3.2. The signature (mental model)

```
map : (T[], (T, index, array) → U) → U[]
```

- Input: array of `T`, a function from `T` to `U`.
- Output: array of `U`, same length.
- The `index` and `array` parameters exist but are rarely needed — when you need them, it's often a sign you want `reduce` instead.

### 1.3.3. Structure preservation — the functor intuition

`map` preserves the **container structure** while transforming the **contents**. The array stays an array, same length, same order — only the values inside change.

This is exactly what "functor" means in the formal sense: a structure-preserving map. You don't need the category theory — just the intuition:

> `map` changes what's *inside* the box without changing the *box itself*.

This intuition extends beyond arrays:
- `Promise.then(fn)` — maps the resolved value, preserves the Promise container.
- `Optional/Maybe.map(fn)` — maps the value if present, preserves the Some/None structure.
- Streams, trees, any "container with contents" can have a `map` that preserves structure.

We'll formalize this in chunk 7 (algebraic structure). For now: `map` = transform contents, preserve container.

### 1.3.4. What `map` is NOT for

`map` is for **transformation** — producing a new value from each element. It's not for side effects.

```js
// BAD — using map for side effects, discarding the result
users.map(u => console.log(u.name));  // L1 — returns [undefined, undefined, ...]

// GOOD — use forEach for side effects
users.forEach(u => console.log(u.name));  // L2 — returns undefined, intent is clear
```

Why this matters: `map` *allocates a new array*. If you don't use the return value, you're paying for an allocation you throw away. More importantly, `map` signals "I'm producing a transformed collection" — using it for side effects misleads the reader.

### 1.3.5. `map` as a `for` loop

Every `map` is equivalent to:

```js
// map version
const result = arr.map(fn);           // L1

// equivalent for loop
const result = [];                    // L2
for (let i = 0; i < arr.length; i++) {  // L3
  result.push(fn(arr[i], i, arr));   // L4
}                                     // L5
```

The `map` version communicates the shape guarantee (same length, 1-to-1) in the method name. The loop version requires reading the body to confirm the same thing.

### 1.3.6. Python parallel

`map(fn, iterable)` in Python — same concept, but returns a lazy iterator (not a list). In practice, Python style prefers list comprehensions: `[fn(x) for x in iterable]`. JS has no comprehension syntax — `map` is the idiomatic equivalent.

