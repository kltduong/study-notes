# 1. Core Iteration Abstractions

> **TL;DR:** `map`, `filter`, and `reduce` are three structural shapes for iterating arrays. `map` = 1-to-1 transform (same length, different values). `filter` = subset selection (≤ length, same values). `reduce` = many-to-one aggregation (collapses to anything). `reduce` is the most general — it can implement the other two. Prefer named methods for self-documenting shape; fall back to loops for early exit or mixed-shape single-pass logic.

## 1.1. The three shapes

Every array iteration falls into one of three structural categories, distinguished by the input→output relationship:

| Shape | Method | Signature (conceptual) | Size relationship | Per-element role |
|-------|--------|------------------------|-------------------|-----------------|
| **1-to-1 transform** | `map` | `T[] → U[]` | `output.length === input.length` | Each output derived from exactly one input |
| **Subset selection** | `filter` | `T[] → T[]` | `output.length ≤ input.length` | Each element independently kept or discarded |
| **Aggregation** | `reduce` | `T[] → U` | Collection → single value | All elements contribute to one accumulated result |

### 1.1.1. Hierarchy

`reduce` is the universal fold — it can implement both `map` and `filter`. The reverse isn't true. The specialized methods exist because **constraints communicate intent**: "this is a map" tells the reader more than "this is a reduce that happens to produce a same-length array."

## 1.2. `map` — structure-preserving transformation

**Contract:** output array has the same length as input. Element `i` of output is derived solely from element `i` of input. Each callback invocation is independent.

```js
const nums = [1, 2, 3, 4];
const doubled = nums.map(x => x * 2);  // [2, 4, 6, 8]
```

**The functor intuition:** `map` changes what's *inside* the container without changing the container itself — same length, same order, only values differ. The type can change (`T[] → U[]` where `T ≠ U`) — the structural guarantee is about the array, not the element types.

This intuition extends beyond arrays: `Promise.then(fn)` maps the resolved value preserving the Promise container. Any "mappable container" follows the same pattern (formalized in chunk 7 as functors).

**Not for side effects.** `map` allocates a new array. If you discard the return value, use `forEach` instead — it signals "side effects only."

> **Aside — Python parallel.** `map(fn, iterable)` returns a lazy iterator. Idiomatic Python prefers list comprehensions: `[fn(x) for x in iterable]`. JS has no comprehension syntax — `.map()` is the idiomatic equivalent.

## 1.3. `filter` — subset selection

**Contract:** output contains only elements from the input (unchanged), in the same relative order, where the predicate returned truthy. `output.length ≤ input.length`.

```js
const nums = [1, 2, 3, 4, 5, 6];
const evens = nums.filter(x => x % 2 === 0);  // [2, 4, 6]
```

The callback is a **predicate** — it answers "keep or discard?" per element. Compare: `map` callback answers "what should this become?"; `filter` callback answers "should this survive?"

### 1.3.1. The `Boolean` trick

```js
["hello", "", "world", null, 0, "foo"].filter(Boolean)
// → ["hello", "world", "foo"]
```

`Boolean` as predicate filters out all falsy values. **Gotcha:** this removes `0` and `""` which might be valid data. For null-only filtering: `items.filter(x => x != null)`.

### 1.3.2. Filter does NOT transform

The predicate's return value is used as a boolean decision, not as the output element. `words.filter(w => w.toUpperCase())` keeps all truthy strings *unchanged* — it doesn't uppercase them.

### 1.3.3. Shared references, not copies

"Elements pass through unchanged" means **references pass through**. `filter` creates a new array (new container), but elements inside are shared with the original — same heap objects.

```js
const data = [{ name: "Alice", age: 30 }, { name: "Bob", age: 17 }];
const adults = data.filter(u => u.age >= 18);  // [{ name: "Alice", age: 30 }]
adults[0].name = "ALICE";
console.log(data[0].name);  // "ALICE" — same object
```

General rule: array methods create new arrays (new containers) but don't clone elements (contents). Applies to `map` too when the callback returns the same object reference.

## 1.4. `reduce` — the general fold

**Contract:** collapses an array into a single value by threading an accumulator through each element. Output can be anything.

```js
const sum = [1, 2, 3, 4].reduce((acc, x) => acc + x, 0);  // 10
```

**Mental model — a `for` loop with explicit state:**

```js
let acc = init;
for (let i = 0; i < arr.length; i++) {
  acc = fn(acc, arr[i], i, arr);  // callback's return REPLACES acc
}
return acc;
```

The callback's return value becomes the next accumulator. Forgetting to return → `acc` becomes `undefined` next iteration.

### 1.4.1. Implementing map and filter via reduce

```js
// map via reduce
arr.reduce((acc, x) => [...acc, fn(x)], []);

// filter via reduce
arr.reduce((acc, x) => pred(x) ? [...acc, x] : acc, []);

// filter + map fused
arr.reduce((acc, x) => pred(x) ? [...acc, fn(x)] : acc, []);
```

### 1.4.2. Always provide an initial value

Without one, `reduce` uses `arr[0]` as the initial accumulator and starts from `arr[1]`. On an empty array → `TypeError`. The explicit initial value handles empty arrays, makes the type clear, and avoids off-by-one confusion.

> 🔖 **Later (chunk 7):** The initial value is the *identity element* of the operation — `0` for `+`, `""` for string concat, `[]` for array concat. This is the monoid pattern.

## 1.5. Composition — chaining the three

### 1.5.1. Method chaining as a pipeline

```js
const orders = [
  { id: 1, total: 250, status: "shipped" },
  { id: 2, total: 50, status: "pending" },
  { id: 3, total: 300, status: "shipped" },
  { id: 4, total: 120, status: "shipped" },
];

orders
  .filter(o => o.status === "shipped")   // subset: 3 orders
  .map(o => o.total)                     // transform: [250, 300, 120]
  .reduce((sum, t) => sum + t, 0);       // aggregate: 670
```

Read as a data pipeline: select → transform → aggregate. Each method name declares the structural shape at that stage.

### 1.5.2. Filter early, map late

**Narrow the dataset as early as possible.** Filter before map means:
1. **Performance** — map processes fewer elements.
2. **Information preservation** — map can destroy structure (objects → numbers). Filtering after map may lose access to fields needed for the predicate.

Default ordering: `filter` → `map` → `reduce`, unless the transform is needed for the filter condition.

### 1.5.3. Intermediate arrays

Each `.filter()` and `.map()` allocates an intermediate array. Almost never matters for typical application code. Matters for 100k+ elements in hot paths.

**Fusing with reduce** eliminates intermediates but loses self-documenting shape names. Modern default: prefer chaining for clarity; fuse only when profiling shows a real bottleneck.

> 🔖 **Later (chunk 4):** Transducers formalize fusion — compose map/filter logic without intermediate arrays while keeping operations named and separate.

## 1.6. Relationship to `for` loops

### 1.6.1. What named methods give you

| Gain | Why |
|------|-----|
| Shape declaration | Method name = structural guarantee (same length / subset / collapse) |
| No mutation | Result is a new value; no `let` + `push` |
| Composability | Methods chain into pipelines |
| Single-responsibility | Each callback does one thing |

### 1.6.2. What loops give you

| Gain | Why |
|------|-----|
| Early exit | `break`/`continue` — not available in map/filter/reduce |
| Single-pass multi-operation | Filter + transform + accumulate in one pass |
| Index manipulation | Skip, look-ahead, variable step |
| Async iteration | `for await...of` works; `.map(async fn)` gives unresolved promises |

### 1.6.3. Decision framework

Named method fits the shape? → Use it. Need early exit? → `find`/`some`/`every` or `for...of`. Need mixed-shape single-pass? → `for` loop (or fused `reduce` if expression form needed). Default: start with named methods, fall back to loops when they can't express the operation cleanly.

### 1.6.4. Loop forms

Prefer `for...of` — gives values directly, supports `break`/`continue`/`await`. Use `for...in` only for object key enumeration (never arrays — yields string indices, includes inherited properties). Classic `for (let i = 0; ...)` for index manipulation or when you need the numeric index.

### 1.6.5. `forEach`

Array method equivalent of a side-effect loop. Returns `undefined`. No `break`, no `await`. Use when you want side effects over each element without early exit; otherwise plain `for...of` is simpler.

## 1.7. Quick reference

- **Three shapes** — map (1-to-1, same length), filter (subset, ≤ length, elements unchanged), reduce (many-to-one, output is anything).
- **Reduce is universal** — can implement map and filter; the reverse isn't true.
- **Filter passes references** — new array, shared elements. Mutation through either reference is visible.
- **Filter early, map late** — narrow first for performance and information preservation.
- **Intermediate arrays** — each chain step allocates one. Almost never a real cost; fuse only when profiled.
- **Named methods vs loops** — methods declare shape; loops offer early exit, single-pass fusion, and async iteration.
