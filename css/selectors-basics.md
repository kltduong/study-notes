# Selector Basics

**TL;DR**

- Five atoms: **type** (`p`), **class** (`.note`), **id** (`#x`), **attribute** (`[type=email]`), **universal** (`*`).
- Combinators describe DOM *relationships* between simple selectors: space (descendant), `>` (child), `+` (adjacent sibling), `~` (general sibling).
- Three ways to combine selectors, easy to mix up:
  - **No separator** → same element, AND (`a.btn.primary`).
  - **Combinator** → relationship between elements (`nav > a`).
  - **Comma** → OR, a selector list (`h1, h2, h3`).
- Pseudo-**class** `:` = state/position (`:hover`, `:first-child`). Pseudo-**element** `::` = styling part of an element that isn't a DOM node (`::before`, `::placeholder`).

---

## The five building blocks

| Kind | Syntax | Matches |
|---|---|---|
| Type | `p` | every `<p>` |
| Class | `.note` | elements with that class |
| ID | `#header` | the element with that id |
| Attribute | `[type="email"]` | elements whose attribute matches |
| Universal | `*` | everything |

Everything more complex is these plus combinators and pseudo-classes.

## Combinators — reading the DOM tree

Written **between** two simple selectors:

| Combinator | Example | Meaning |
|---|---|---|
| (space) | `article p` | `p` anywhere inside an `article` |
| `>` | `article > p` | `p` that is a **direct child** of `article` |
| `+` | `h2 + p` | `p` **immediately after** an `h2` (same parent) |
| `~` | `h2 ~ p` | any `p` **after** an `h2` (same parent) |

```html
<article>
  <h2>Title</h2>
  <p>First</p>        <!-- h2 + p  AND  h2 ~ p -->
  <p>Second</p>       <!-- only h2 ~ p -->
  <section>
    <p>Nested</p>     <!-- article p, but NOT article > p -->
  </section>
</article>
```

Left-to-right in a selector = down-and-across in the tree.

## Compound vs complex vs list

The piece that trips people up:

- **Compound selector** — simple selectors stuck together, no separator: `a.btn.primary`. All must match the **same element** (AND).
- **Complex selector** — simple/compound selectors joined by **combinators**: `nav > a.btn`. Describes a relationship.
- **Selector list** — comma-separated: `h1, h2, h3`. Matches if any applies (OR).

Mental shortcut: **no separator = AND on one element; combinator = relationship; comma = OR**.

## Pseudo-classes vs pseudo-elements

One colon vs two — they are different things.

- **Pseudo-class** (`:`) — matches elements based on something not in the markup: state or position. `:hover`, `:focus`, `:checked`, `:disabled`, `:first-child`, `:nth-child(2n)`, `:not(...)`, `:is(...)`, `:where(...)`, `:has(...)`.
- **Pseudo-element** (`::`) — styles a *part* of an element that isn't a DOM node: `::before`, `::after`, `::first-line`, `::first-letter`, `::placeholder`, `::selection`, `::marker`.

Legacy form `:before` (one colon) still works; the modern correct form is `::before`.

## What counts as "the same element"?

A common confusion with compound selectors: `h2.title` means an `<h2>` that *also* has class `title`. It is still one element; you're piling conditions on it.

```css
h2.title        /* one element, two conditions (tag + class) */
h2 .title       /* two elements — the .title is a descendant of h2 */
```

The space changes the meaning entirely.

## See also

- [specificity-and-cascade.md](./specificity-and-cascade.md) — how conflicts between matching rules are resolved.
- [modern-selectors.md](./modern-selectors.md) — `:has()`, `:is()`, `:where()`, `:not()`.
- [attribute-and-structural.md](./attribute-and-structural.md) — attribute operators and structural pseudo-classes in depth.
