# Attribute & Structural Selectors

**TL;DR**

- Attribute operators: `=` exact, `~=` whitespace-word, `|=` value or `value-*`, `^=` starts with, `$=` ends with, `*=` contains. Append `i` for case-insensitive.
- `:nth-child(n)` counts **all** siblings. `:nth-of-type(n)` counts only siblings with the same tag. They are **not** interchangeable.
- `p:first-child` means "a `p` that is also the first child" — often matches nothing. Usually you want `p:first-of-type`.
- `:empty` is strict: any whitespace inside disqualifies an element.
- `:nth-child(n of selector)` is the modern way to say "the Nth matching sibling".

---

## Attribute selectors

| Syntax           | Matches when attribute value…                           |
| ---------------- | ------------------------------------------------------- |
| `[attr]`         | exists (any value)                                      |
| `[attr=value]`   | equals exactly                                          |
| `[attr~=value]`  | contains `value` as a **whitespace-separated word**     |
| `[attr\|=value]` | equals `value` or starts with `value-` (language codes) |
| `[attr^=value]`  | **starts with** `value`                                 |
| `[attr$=value]`  | **ends with** `value`                                   |
| `[attr*=value]`  | **contains** `value` as a substring                     |

```css
a[href^="https"] {
  /* external, secure */
}
a[href$=".pdf"]::after {
  content: " 📄";
}
img[alt=""] {
  /* empty alt — decorative */
}
img:not([alt]) {
  outline: 3px solid red; /* missing alt = bug */
}
[class*="icon-"] {
  /* any icon-* class */
}
[lang|="en"] {
  /* en, en-US, en-GB */
}
```

### The `~=` gotcha

`[class~="btn"]` on `class="btn-primary"` does **not** match — `btn-primary` is a single word. That's also how `.btn` the class selector is defined under the hood; it's sugar for `[class~="btn"]`.

### Case sensitivity

Attribute values are case-sensitive by default in HTML (except for known case-insensitive attributes like `type`). Add `i` for case-insensitive, `s` for forced case-sensitive (rarely needed):

```css
input[type="email" i]   /* matches type="EMAIL", "Email", etc. */
a[href$=".PDF" i]       /* catches .pdf, .PDF, .Pdf */
```

## `:nth-child` vs `:nth-of-type`

The classic trap.

### `:nth-child(n)` — position among **all** siblings

Counts every child, regardless of tag.

```html
<article>
  <h2>Title</h2>
  <!-- child 1 -->
  <p>First</p>
  <!-- child 2 -->
  <p>Second</p>
  <!-- child 3 -->
</article>
```

```css
p:nth-child(2) {
  /* "First" — child #2 AND a p */
}
p:nth-child(1) {
  /* nothing — child #1 is an h2 */
}
```

Read it as: "element that **is** a `p` **and** is the Nth child of its parent". Both must hold.

### `:nth-of-type(n)` — position among siblings **of the same tag**

```css
p:nth-of-type(1) {
  /* "First" — 1st p, ignoring the h2 */
}
p:nth-of-type(2) {
  /* "Second" */
}
```

### The formula `an + b`

`n` iterates over 0, 1, 2, ...

| Formula        | Matches        |
| -------------- | -------------- |
| `2n` / `even`  | 2nd, 4th, 6th… |
| `2n+1` / `odd` | 1st, 3rd, 5th… |
| `3n`           | every 3rd      |
| `3n+1`         | 1st, 4th, 7th… |
| `-n+3`         | first 3        |
| `n+4`          | 4th onwards    |

```css
li:nth-child(odd) {
  background: #f5f5f5;
}
tr:nth-child(3n + 1) {
  border-top: 2px solid;
}
li:nth-child(-n + 3) {
  font-weight: bold; /* first three */
}
```

### Modern: `:nth-child(n of selector)`

Old limitation: no way to say "the 2nd `.featured` item". Now:

```css
li:nth-child(2 of .featured) {
  /* 2nd li with class .featured */
}
```

**Not** the same as `.featured:nth-child(2)`, which requires the 2nd child overall to _also_ be `.featured`.

## `:first-child`, `:last-child`, `:only-child`

Shorthands. Same "all siblings" counting as `:nth-child`:

```css
p:first-child    /* a p that is ALSO the first child — often matches nothing */
p:first-of-type  /* the first p among its siblings — usually what you want */
```

The failure mode: `article p:first-child` expecting "the first paragraph", but the article starts with `<h2>` — nothing matches.

## `:empty`

Matches an element with **no children at all** — including no text, no whitespace. `<p></p>` matches; `<p> </p>` does **not**.

```css
.notice:empty {
  display: none;
}
```

## `:only-of-type`

Sole element of its tag among siblings:

```css
.tag:only-child {
  margin: 0; /* no siblings, no gap */
}
section:only-of-type {
  /* the only <section> in its parent */
}
```

## See also

- [selectors-basics.md](./selectors-basics.md) — the pseudo-class category these live in.
- [specificity-cascade.md](./specificity-cascade.md) — these pseudo-classes count toward slot `b`.
