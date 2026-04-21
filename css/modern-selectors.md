# Modern Selectors — `:has()`, `:is()`, `:where()`, `:not()`

**TL;DR**

- `:has()` is the long-missing **parent selector**: `A:has(B)` matches `A` if `B` matches inside (or, with combinators, relative to) `A`. Widely supported since late 2023.
- `:is(X)` / `:not(X)` / `:has(X)` contribute the **highest specificity among their arguments**. `:where(X)` always contributes **0**.
- `:where()` is the modern escape hatch for low-specificity defaults (resets, libraries).
- Inside `:has()`: no nested `:has()`, no pseudo-elements.

---

## `:has()` — parent selector

`A:has(B)` matches `A` when `B` matches something *inside* it, or — with combinators — relative to it.

```css
figure:has(figcaption)          { margin-bottom: 2rem; }
label:has(input:invalid)        { color: red; }
.card:not(:has(img))            { padding: 2rem; }
h2:has(+ p)                     { margin-bottom: 0; }
form:has(:checked)              { border-color: green; }
body:has(.modal[open])          { overflow: hidden; }
```

The last one is the game-changer: disabling page scroll when a modal is open used to require JS toggling a class on `<body>`. Now it's one CSS rule.

### Gotchas

- **No nested `:has()`** — `:has(:has(...))` is invalid (spec restriction for implementability).
- **No pseudo-elements inside `:has()`** — `:has(::before)` is invalid.
- **Performance**: `:has()` reverses the normal "match from descendant upward" model — the engine must re-check ancestors when descendants change. Modern engines optimize it well; don't preemptively avoid it, but don't sprinkle `body:has(...)` rules carelessly either.

## `:is()` — shorthand for selector lists

Takes a selector list and matches if any applies. Equivalent to duplicating the outer selector.

```css
:is(h1, h2, h3) + p   /* = h1 + p, h2 + p, h3 + p */
```

Specificity = the highest specificity among its arguments. The `:is()` itself contributes nothing.

## `:where()` — `:is()` that always scores 0

Same matching behavior as `:is()`, but its specificity contribution is always **0**. This makes it perfect for styles you want to be easily overridable:

```css
:where(ul, ol) { padding-inline-start: 1rem; }  /* consumers can override with any selector */
```

Common pattern in reset stylesheets and design systems: wrap everything in `:where(...)` so consumers never fight the cascade.

## `:not()` — negation

Takes a selector list. Matches elements **not** matching any of them.

```css
button:not(.primary, .danger) { /* neutral buttons */ }
li:not(:last-child)           { border-bottom: 1px solid; }
```

Specificity = the highest specificity among its arguments (same rule as `:is()`/`:has()`).

## Specificity at a glance

| Function | Contributes |
|---|---|
| `:is(X)` | max(specificity of X) |
| `:not(X)` | max(specificity of X) |
| `:has(X)` | max(specificity of X) |
| `:where(X)` | **0** |

```css
:is(#sidebar, .menu) a     /* (1,0,1) */
:where(#sidebar, .menu) a  /* (0,0,1) */
:not(.foo, #bar)           /* (1,0,0) */
```

## When to reach for which

- **`:is()`** — DRY up a list of selectors sharing the same trailing pattern.
- **`:where()`** — you want the rule to lose almost every override fight on purpose.
- **`:not()`** — carve out exceptions from a broader match.
- **`:has()`** — style a parent/ancestor based on its contents, or style an element based on a sibling relationship you couldn't express before.

## See also

- [specificity-and-cascade.md](./specificity-and-cascade.md) — how the specificity numbers combine into the `(a,b,c)` tuple.
- [selectors-basics.md](./selectors-basics.md) — combinators, which also appear inside `:has()`.
