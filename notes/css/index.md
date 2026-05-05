# CSS Selectors

How the browser picks which elements a rule applies to, and which rule wins when several match.

## The mental model

A **selector** is a pattern matched against the DOM. Styling breaks down into two questions:

1. **Which elements does this rule match?** — answered by the selector syntax.
2. **When several rules match the same element, which one wins?** — answered by the cascade (origin, importance, layers, specificity, source order).

Keep these separate. "Why isn't my style applying?" is almost always question 2, not question 1.

## Reading order

Start with basics; cascade is the load-bearing concept. The other two files are reference-flavored — read when you hit the relevant problem.

1. **[selectors-basics.md](./selectors-basics.md)** — building blocks (type, class, id, attribute, pseudo), combinators (`>`, `+`, `~`, space), and the difference between compound selectors, complex selectors, and selector lists.
2. **[specificity-cascade.md](./specificity-cascade.md)** — the full resolution order: origin & importance, cascade layers (`@layer`), the `(a,b,c)` specificity tuple, source order. This is the file to reach for when a style isn't applying.
3. **[modern-selectors.md](./modern-selectors.md)** — `:has()` (the parent selector), `:is()`, `:where()`, `:not()`, and how each affects specificity.
4. **[attr-structural.md](./attr-structural.md)** — attribute operators (`^=`, `$=`, `*=`, `~=`, `|=`), the `nth-child` vs `nth-of-type` trap, `:empty` and friends.

## Cross-cutting ideas

### Specificity shows up everywhere

Every pseudo-class, attribute selector, and logical function contributes to the `(a,b,c)` tuple in a specific way. A quick reference:

| Thing                                        | Slot           | Notes                                              |
| -------------------------------------------- | -------------- | -------------------------------------------------- |
| `#id`                                        | a              |                                                    |
| `.class`, `[attr]`, `:hover`, `:nth-child()` | b              | pseudo-**classes**                                 |
| `tag`, `::before`                            | c              | pseudo-**elements**                                |
| `*`, combinators (`>`, `+`, `~`, space)      | —              | nothing                                            |
| `:is(X)`, `:not(X)`, `:has(X)`               | highest of `X` | the wrapper adds nothing                           |
| `:where(X)`                                  | **0**          | always — escape hatch for low-specificity defaults |

### Read selectors structurally, not word-by-word

Given `nav ul > li.active a[href^="/"]`:

- Whitespace between simple selectors is the **descendant** combinator.
- Combinators (`>`, `+`, `~`) describe DOM _relationships_.
- No space (`li.active`, `a[href^="/"]`) means **same element, multiple conditions** (AND).
- A comma would mean **OR**.

Being fluent at parsing selectors this way makes the rest trivial.

### The "is my style applying?" debugging flow

1. DevTools → Styles/Computed panel. The winning rule is at the top; losers are struck through.
2. Check for `!important` and `@layer` badges before blaming specificity.
3. If it is specificity, add a class to raise it naturally — don't reach for `!important`.
4. Persistent cascade fights are a signal to adopt `@layer`, not to escalate.

## Related topics

- [client-side/](../client-side/) — the DOM that selectors target.
