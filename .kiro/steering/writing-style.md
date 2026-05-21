---
inclusion: always
---

# Writing Style & Formatting

Rules for writing and formatting study notes. Applies to all notes in both `courses/` and `notes/`.

## Before writing

- State assumptions explicitly. If uncertain, ask.
- Multiple interpretations → present them, don't pick silently.

## Content rules

- Capture reasoning, motivation, and non-obvious details — not textbook definitions.
- Include concrete examples, commands, and gotchas from the session. Three roles for examples, all welcome:
  - **Anchor** — a small real example that introduces a mechanism before abstracting.
  - **Bug demo** — a minimal snippet showing the concrete failure of an alternative.
  - **Worked synthesis** — a single annotated example that ties the pieces together after they've been explained.
- Preserve the mental model that emerged.
- Link related notes with relative links (same folder only).
- Short TL;DR at the top of each note.
- **Quick reference section at the end.** Close each note with a `## Quick reference` section — a tight bullet list (3–6 items) summarizing the note's key mechanisms in one line each. Format: `- **Label** — one-sentence takeaway.` Purpose: future-self skims this first on revisit without re-reading the full note.
- **Right tool for the job.** When presenting multiple syntactic forms or approaches that achieve similar results, state explicitly which to reach for *in which situation* — performance characteristics, readability tradeoffs, semantic fit, engine optimization differences. If one is the modern default for most new code, say so. If an older form is still the right choice in specific contexts (e.g. `for` loop when you need early `break` vs `.forEach` for side-effects vs `.map` for transforms), explain when each wins. If a form exists only for legacy compatibility, say so explicitly (e.g. "legacy — use X instead"). Don't leave the reader to infer the recommendation from mechanism alone.

  This applies broadly — not just to picking between sibling syntactic forms, but to **any language feature with tradeoffs**: capabilities that enable cool patterns but smell in everyday code (method borrowing, prototype mutation, mixins via `Object.assign`, run-time prototype reassignment, etc.). Whenever the mechanism teaches a *capability*, pair it with the *judgment* — a "status / when to use" treatment so future-self doesn't blindly apply a powerful-but-niche tool. A short capabilities table with a "smell vs OK in" column is the canonical shape for this when several related capabilities are in play.

## Style rules

**Top-down clarity** — at every point, the reader understands using only what came before.

- Define jargon on first use, including in the TL;DR.
- No unexplained forward references. If deferring detail, give enough plain-language context now + a forward link.
- Each idea explained once. Pick the best section for the full treatment; link from elsewhere.
- Don't restate before forwarding. One statement + forward link is enough.
- **Deepen, don't stack.** When a clarification expands an existing argument, weave it into the relevant section — don't add a parallel subsection. Multiple siblings each answering a variant of the same "why?" question → merge into one linear chain.
- **If a prerequisite has its own chunk, one-line + forward link** — don't mini-explain it inline.
- **Show the bug, not the label.** When explaining why a design choice beats an alternative, demonstrate the concrete failure (small example, real symptom). "Silent bugs" as a phrase is weaker than a three-line snippet showing `if (!user)` firing wrongly.
- **Worked example for synthesis.** After the pieces of a mechanism have been explained individually, a single annotated example that traces multiple cases through the same mechanism cements the mental model. Inline comments trace the mechanism step-by-step (which pointer, which state, which transition) so the reader can single-step through. Place it _after_ the sections it consolidates, not before — it's a capstone, not an anchor. Keep to one per note; more means the individual sections aren't landing on their own. Annotate selectively — every line that introduces a new case, not every line.
- **List only independent items.** When enumerating, check whether any item is a consequence of another in the list. If so, drop or fold it as a sub-point. N items implies N independent points — don't inflate with derivable ones.
- **Asides in blockquotes.** Tangential content that's useful but sits outside the main derivation — terminology sidebars, meta-notes, historical context, formal-layer remarks — goes in a `> **Aside —** …` blockquote rather than its own heading. The note should read top-to-bottom without the asides and still land. Guardrails:
  - **Don't aside a prerequisite.** If the next section builds on the content, it's a step, not an aside.
  - **Don't stack asides.** Two in a row means the main thread is too thin — rework the flow instead.
  - **One role per blockquote prefix.** `> ⚠️` is reserved for critical-lens flags on source content; `> **Aside —**` is for tangents; `> 🔖 Later:` flags an explicit rabbit hole for future exploration. Don't mix.

**Tone:** Concise — notes are for future-self, not strangers. No filler. Match existing note style in the folder.

## Formatting

Markdown with headings, fenced code blocks (with language tags), tables, and mermaid diagrams.

### Diagrams

For flow and relationship diagrams, **use mermaid — not ASCII/Unicode box-drawing art.** Mermaid renders as graphs with arrows; box-drawing lines (`─ │ ┌ └`) can't express that.

**Exception:** inside code blocks, box-drawing tree characters (`├── └──`) are fine for showing hierarchical _data-structure layouts_ (fields of a record, directory trees, object layouts) — mermaid is clunky for that shape. Plain text in code blocks for pseudo-code/asm is also fine.

**Mermaid node styling:** dark fills with white text (e.g. `fill:#46c,stroke:#fff,color:#fff`) for dark-mode legibility. Avoid light pastel fills with dark text.

### Mermaid label clipping

Subgraph titles and edge labels clip when they exceed the rendered box width — mermaid doesn't wrap text. To handle long explanations:

- **Keep diagram labels short.** Use abbreviations or terse phrases in subgraph titles, node labels, and edge labels.
- **Offload detail to a markdown legend below the diagram.** The legend is regular prose — no length constraints.
- **Mark labels with `†`** in the diagram to signal "see legend below." Start the legend block with `**† Legend:**` so the dagger in the diagram points the reader to the matching dagger in the prose.
- **Abbreviations line.** After the legend bullets, add a one-liner expanding any abbreviations used in node/edge labels (e.g. `**Abbreviations:** LexEnv = LexicalEnvironment, ER = Environment Record`).

Principle: the diagram handles spatial relationships and compact labels; the prose handles explanation. The `†` bridges the two.

## Surgical edits

- Touch only what changed. Don't reformat adjacent sections.
- If a note grows unwieldy, suggest splitting — don't silently restructure.
