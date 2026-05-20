# 1. Core Iteration Abstractions — Draft

## 1.1. Plan (teaching order)

- [x] Three structural shapes — the taxonomy (filter: subset, map: 1-to-1, reduce: many-to-one)
- [x] `map` deep dive — structure preservation, the functor intuition
- [x] `filter` deep dive — predicate as selector, Boolean coercion trick
- [x] `reduce` intro — the accumulator loop, why it's the most general
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

---

## 1.4. `filter` — subset selection

### 1.4.1. The contract

`filter` makes a different promise: **the output contains only elements from the input (unchanged), in the same relative order, where the predicate returned truthy.**

```js
const nums = [1, 2, 3, 4, 5, 6];              // L1
const evens = nums.filter(x => x % 2 === 0);  // L2 — [2, 4, 6]
```

Key constraints:
- `output.length ≤ input.length` (can be 0 if nothing passes, can equal input if everything passes).
- Elements are **not transformed** — they pass through as-is. The output type is `T[]`, same as input.
- Order is preserved — relative positions don't change.

### 1.4.2. The signature

```
filter : (T[], (T, index, array) → boolean) → T[]
```

The callback is a **predicate** — a function that returns true/false (or truthy/falsy). It answers one question per element: keep or discard?

Compare with `map`:
- `map` callback: "what should this element *become*?" → produces the output value.
- `filter` callback: "should this element *survive*?" → produces a yes/no decision.

### 1.4.3. The `Boolean` trick

From the teaser:

```js
["hello", "", "world", null, 0, "foo"].filter(Boolean)
// → ["hello", "world", "foo"]
```

`Boolean` is a function — when called with a value, it returns `true` for truthy, `false` for falsy. Passing it as the predicate filters out all falsy values (`""`, `null`, `0`, `undefined`, `NaN`, `false`).

This works because `filter` only cares about the truthiness of the return value. `Boolean` is just a convenient predicate that maps directly to JS's truthy/falsy rules.

**Gotcha:** This removes `0` and `""` — which might be valid data. Use `Boolean` only when you genuinely want to strip all falsy values, not just `null`/`undefined`. For null-only filtering:

```js
items.filter(x => x != null)  // keeps 0, "", false — removes only null/undefined
```

### 1.4.4. `filter` does NOT transform

A common mistake — trying to filter and transform in one step:

```js
// WRONG mental model: "filter to the uppercase versions"
const upper = words.filter(w => w.toUpperCase());  // L1 — BUG
```

L1 doesn't filter by uppercase-ness — it filters by truthiness of `w.toUpperCase()`. Since any non-empty string is truthy, this keeps all non-empty strings *unchanged* (not uppercased). The return value of the predicate is used as a boolean decision, not as the output element.

If you need to filter AND transform, you need both:
```js
words.filter(w => w.length > 3).map(w => w.toUpperCase())
```

Or `reduce` (which can do both in one pass — next course chunk: "Reduce deep dive").

### 1.4.5. Shared references, not copies

"Elements pass through unchanged" means **references pass through** — not deep copies. `filter` creates a new array (new container), but the elements inside are shared with the original.

```js
const data = [{ name: "Alice", age: 30 }, { name: "Bob", age: 17 }];  // L1
const adults = data.filter(u => u.age >= 18);  // L2 — [{ name: "Alice", age: 30 }]
adults[0].name = "ALICE";                       // L3 — mutates the shared object
console.log(data[0].name);                      // L4 — "ALICE" (same heap object)
```

The `for` loop equivalent makes this visible: `result.push(arr[i])` copies the reference from `arr[i]` into the new array's slot. Both arrays hold references to the same objects. Mutation through either reference is visible through the other.

This applies to `map` too when the callback returns the same object reference (rare but possible). The general rule: array methods create **new arrays** (new containers), but don't clone the **elements** (contents).

### 1.4.6. `filter` as a `for` loop

```js
// filter version
const result = arr.filter(pred);              // L1

// equivalent for loop
const result = [];                            // L2
for (let i = 0; i < arr.length; i++) {        // L3
  if (pred(arr[i], i, arr)) {                 // L4
    result.push(arr[i]);                      // L5 — element passes through unchanged
  }
}
```

The loop makes visible what `filter` guarantees: elements are pushed as-is (L5), only the `if` gate (L4) varies.

### 1.4.7. Python parallel

`filter(fn, iterable)` exists but is rarely used — Python prefers `[x for x in iterable if pred(x)]`. Same semantics: subset selection, elements unchanged, order preserved.



---

## 1.5. `reduce` — the general fold

### 1.5.1. The contract

`reduce` makes the weakest structural promise: **it collapses an array into a single value by threading an accumulator through each element.** The output can be anything — number, string, object, array, function.

```js
const nums = [1, 2, 3, 4];                          // L1
const sum = nums.reduce((acc, x) => acc + x, 0);    // L2 — 10
```

At each step: `callback(accumulator, currentElement)` → new accumulator. The final accumulator is the result.

### 1.5.2. The signature

```
reduce : (T[], (Acc, T, index, array) → Acc, Acc) → Acc
```

- **Accumulator (`Acc`)** — the "state so far." Starts at the initial value, gets replaced each iteration by the callback's return.
- **Callback** — takes current accumulator + current element, returns the *next* accumulator.
- **Initial value** — the accumulator before the first element is processed.

### 1.5.3. The accumulator loop (mental model)

`reduce` is a `for` loop with an explicit state variable:

```js
// reduce version
const result = arr.reduce(fn, init);          // L1

// equivalent for loop
let acc = init;                               // L2
for (let i = 0; i < arr.length; i++) {        // L3
  acc = fn(acc, arr[i], i, arr);              // L4 — acc is REPLACED each iteration
}                                             // L5
const result = acc;                           // L6
```

The key insight: **the callback's return value becomes the next accumulator.** If you forget to return, `acc` becomes `undefined` on the next iteration — a common bug.

### 1.5.4. Why it's the most general

`map` and `filter` are both special cases of `reduce`:

```js
// map via reduce
arr.reduce((acc, x) => [...acc, fn(x)], []);          // L1

// filter via reduce
arr.reduce((acc, x) => pred(x) ? [...acc, x] : acc, []);  // L2
```

`reduce` can do anything a loop can — it's the universal iteration primitive. `map` and `filter` are `reduce` with constraints:
- `map` = reduce where you always push exactly one transformed element.
- `filter` = reduce where you conditionally push the original element.

The specialized methods exist because constraints communicate intent. "This is a map" tells the reader more than "this is a reduce that happens to produce a same-length array."

### 1.5.5. The initial value question

What happens without an initial value?

```js
[1, 2, 3].reduce((acc, x) => acc + x);     // L1 — works: uses arr[0] as init, starts from arr[1]
[].reduce((acc, x) => acc + x);             // L2 — TypeError: reduce of empty array with no initial value
```

Without an initial value, `reduce` uses the first element as the accumulator and starts iteration from the second. On an empty array, there's no first element → crash.

**Rule: always provide an initial value.** It makes the type explicit, handles empty arrays gracefully, and avoids the off-by-one confusion of "first element is both accumulator and data." The only exception: when you're guaranteed a non-empty array AND the accumulator type matches the element type (like summing numbers). Even then, the explicit `0` costs nothing and prevents surprises.

> 🔖 **Later (chunk 7 — algebraic structure):** The initial value is the *identity element* of the operation. `0` for addition, `""` for string concat, `[]` for array concat. This isn't coincidence — it's the monoid pattern. We'll formalize it there.

### 1.5.6. Python parallel

`functools.reduce(fn, iterable, initial)` — same semantics. Python style discourages it in favor of explicit loops or comprehensions for readability. JS leans into it more heavily because method chaining (`.filter().map().reduce()`) reads as a pipeline.

