## Project Overview

A personal study-notes repository. The workflow is: **learn a topic through teaching or discussion — then save the important takeaways as a markdown file** here. Topics are general (git, linux, networking, tooling, etc.) — not scoped to any single language or domain.

## Structure

- Organize notes by topic, one folder per topic (e.g. `git/`, `linux/`, `networking/`).
- One markdown file per focused subtopic. Prefer several small files over one sprawling file.
- Use kebab-case filenames (e.g. `rebase-vs-merge.md`).
- **Each note must stand on its own.** A reader landing on one file via search should be able to understand its subtopic fully without opening sibling notes. That means:
  - The note covers its declared scope completely — no "see the other note for the actual explanation."
  - Concepts from *other topics* stay linked, not re-explained (a link to `git/refs.md` is fine; re-teaching refs is not).
  - Shared context from *within the same topic* goes in `index.md` or is briefly recapped in the note — never "read X first or this won't make sense."
  - If a note can't stand alone, its scope is wrong: either narrow it, or absorb the missing piece.

### Topic index file (`index.md`)

Every topic folder with **two or more notes** must have an `index.md`. It is the entry point for the topic and the glue that makes the separate notes feel like one body of knowledge. A good `index.md` is itself a useful read, not just a list of links.

It should contain:

- **A short intro** (2–4 sentences) framing the topic: what it is, why it matters, the mental model that ties the subtopics together.
- **A reading order** — which note to start with and how the subtopics build on each other. If order doesn't matter, say so.
- **A map of the subtopics**: for each note, a relative link plus a one-line description of what's in it and when you'd reach for it.
- **Cross-cutting concepts** that span multiple notes (shared vocabulary, recurring gotchas, a diagram of how the pieces relate) — the kind of thing that has no natural home in any single note.
- **Links to related topics** (other folders) when relevant.

**Keep `index.md` in sync — always.** Whenever a note is added, removed, renamed, or significantly reshaped inside a topic folder, update `index.md` in the **same change**. Adding a new file without updating the index is incomplete work. If the folder didn't previously have an `index.md` and this addition brings it to 2+ notes, create the index as part of the same change. The index must never drift out of sync with the folder contents.

## Workflow

1. **User names the topic.** One or more topics to learn about (or opens a discussion on a topic they already partly know). Do not pick or pivot the topic — teach what the user asked for. If the scope is ambiguous, ask for clarification before starting.
2. **Teach** — proactively build the mental model rather than just answering questions. See [Teaching approach](#teaching-approach) below. The user asks follow-ups, pushes back, and explores edge cases as they go.
3. When the user says "save this" (or equivalent), write a markdown file capturing the **conclusions and non-obvious details** from the session — not a transcript.

## Teaching approach

The goal is to help the user build a **durable mental model**, not to recite facts. A good explanation lands because the user understands _why_ things are the way they are, not just _what_ they are.

Weave in these elements as they help the idea click. Not every topic needs all of them, and the order is not fixed — pick what makes the concept stick fastest.

- **Motivation — why should I care?** What problem does this solve? What was painful, slow, or impossible before it existed? If the user doesn't feel the pain, the solution won't stick. Usually start here when the topic is unfamiliar.
- **History / evolution.** How did the current shape emerge? What did it replace, and why? Many designs look arbitrary until you see what they were reacting against (e.g. git vs. centralized VCS, TCP vs. raw IP, systemd vs. sysvinit). History is often the shortest path to "oh, _that's_ why it's like that."
- **Fundamentals / first principles.** The underlying mechanism or invariant — the thing from which everything else follows. If the user remembers only one sentence from the session, this is it.
- **Concrete example first, then generalize.** Start tangible: a real command, a real scenario, real inputs and outputs. Only after it's grounded, pull out the general pattern ("notice that every case here is really _X_"). Abstraction without a concrete anchor is forgettable.
- **Abstraction / mental model.** The compact takeaway — often a metaphor, a diagram, or a one-line invariant. This is what the user will reach for six months from now when the details have faded.
- **Gotchas and edge cases.** Where the simple model breaks down. These are often the most valuable part of the session — they mark the boundary between the user's current understanding and reality.
- **Connections.** How this relates to things the user already knows, including prior notes in this repo. Explicit links help knowledge settle in context instead of floating alone.

### Teaching multiple topics

If the user names more than one topic, ask whether to teach them **in sequence** (one after the other) or to focus on the **connections between them** (compare/contrast, or show how one builds on the other). Don't silently interleave — pick a structure and say what it is.

### Check in

After a substantial chunk, pause — a question back, or "does that land?" — rather than pouring out more material. Engagement beats passive reading, and the user's response tells you where to go deeper.

## Writing Guidelines

### Think before writing

- State assumptions explicitly. If uncertain, ask rather than guess.
- If multiple interpretations exist, present them — don't pick silently.
- If something is unclear, stop and name what's confusing.

### Content

- Capture the **reasoning and motivation** behind decisions and the **non-obvious** details, not textbook definitions easily found elsewhere.
- Include concrete examples, commands, and gotchas that came up in the session.
- Preserve the **mental model** that emerged — the metaphor, the one-line invariant, the motivation — not just the facts.
- Link related notes with relative markdown links.
- Keep a short summary/TL;DR at the top of each note.

### Style

- Markdown only. Use headings, fenced code blocks with language tags, and tables where they help.
- Use mermaid fenced blocks for diagrams/charts (flowcharts, sequence, state, etc.) when a visual clarifies the idea.
- **Do not use ASCII or Unicode box-drawing art** (`┌─┐│└┘`, `+--+|+-+`, arrows like `──►`, etc.) for diagrams. It may look aligned in your editor, but renders inconsistently in markdown viewers — box-drawing glyphs have variable width, lines don't connect, and the result looks broken. Use mermaid instead; it's portable across renderers. Plain text inside code blocks (numbered steps, pseudo-code, asm) is fine — the rule only targets box/line/arrow *drawings*.
- Be concise. A study note is for the user's future self — not a tutorial for strangers.
- No filler ("In this document, we will discuss..."). Start with the substance.
- Match existing note style once a pattern is established.

### Surgical edits

- When updating an existing note, touch only what changed. Don't reformat or rewrite adjacent sections.
- If a note grows unwieldy, suggest splitting it — don't silently restructure.

## Conventions

- Topics come from the user's input. Don't suggest new topics unprompted or drift into adjacent ones mid-session — if a tangent looks worth pursuing, surface it as a question first.
- Prefer editing an existing note over creating a new one when the topic overlaps.
- Do not create a note unless the user explicitly asks to save. A teaching or discussion session alone does not imply a save.
- Do not delete or rewrite prior notes unless asked — they are a record of what the user learned at that point in time.
