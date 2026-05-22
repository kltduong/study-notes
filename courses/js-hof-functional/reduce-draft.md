# Reduce Deep Dive — Teaching Draft

## Plan (teaching order)

- [ ] **Teaser** — concrete failure motivating accumulator design
- [ ] **Reduce as the universal fold** — formal layer, signature, threading mental model
- [ ] **Accumulator design** — init value declares shape and type; type-uniform vs type-changing folds
- [ ] **Building abstractions from reduce** — map/filter/find/some/every/max/groupBy/partition; the early-exit caveat
- [ ] **Mutate-vs-immutable accumulator** — `[...acc, x]` is O(n²); `acc.push(x)` is O(n); when mutation is safe
- [ ] **Common pitfalls** — forgetting return, missing initial value, shape mismatch, no `break`
- [ ] **`reduceRight` and direction** — when right-fold matters (function composition, right-associative ops)
- [ ] **Worked synthesis** — annotated example tying the pieces together

---

## Chunk opener — teaser

```js
const items = [                                  // L1
  { type: "fruit", name: "apple" },
  { type: "veg",   name: "carrot" },
  { type: "fruit", name: "banana" },
  { type: "veg",   name: "spinach" },
];

const grouped = items.reduce((acc, item) => {    // L2
  acc[item.type].push(item.name);                // L3
  return acc;                                    // L4
}, {});

console.log(grouped);                            // L5
```

**Predict:** what does this output? Why?

*(Reveal pending user prediction.)*
