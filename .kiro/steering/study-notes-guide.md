## Project Overview

A personal study-notes repository — **learn a topic through teaching or discussion, then save the takeaways as a markdown file**. Topics are general (git, linux, networking, tooling, etc.) — not scoped to any single language or domain.

## Structure

- One folder per topic (e.g. `git/`, `linux/`). One markdown file per focused subtopic. Prefer several small files over one sprawling file.
- Kebab-case filenames, **≤ 25 characters** including `.md`. Drop filler words (`and`, `the`, `of`) to stay short.
- **Each folder is self-contained:**
  - **No cross-directory links.** If two folders need the same concept, each explains it independently. Duplication _across_ folders is fine.
  - **No reading across directories.** When working in a folder, only read files inside that folder. Don't read notes from other topic folders for context, calibration, or background — treat each folder as if the others don't exist.
  - **No duplication _within_ a folder.** Each idea explained once; notes link to siblings; `index.md` is the glue.
  - **Each note stands on its own** within its folder. Cover the declared scope fully — no "see the other note for the actual explanation." If a note can't stand alone, its scope is wrong: narrow it, or absorb the missing piece.

### Topic index file (`index.md`)

Required when a folder has **two or more notes**. A good `index.md` is itself a useful read, not just a link list. It should contain:

- **Short intro** (2–4 sentences) framing the topic and the mental model tying subtopics together.
- **Reading order** — which note to start with and how they build on each other (or state that order doesn't matter).
- **Subtopic map**: relative link + one-line description per note.
- **Cross-cutting concepts** that span multiple notes (shared vocabulary, recurring gotchas, how pieces relate).

**Keep `index.md` in sync — always.** Any note added, removed, renamed, or significantly reshaped → update `index.md` in the same change. If the addition brings the folder to 2+ notes, create `index.md` in the same change.

### Course table of contents (`toc.md`)

Created at the start of the [course-guided workflow](#course-guided-workflow). Tracks the **original course structure and progress** — a different job from `index.md` (which maps notes by mental model, not course order).

- One entry per course section/chunk, in course order.
- Each entry: checkbox + section title + brief scope note.
- Mark chunks `[x]` as they are taught during the session.
- `toc.md` is **not** linked from `index.md` — it's a progress artifact, not a study note.

### Quizzes and tests

Quizzes and tests are interactive checkpoints woven into the toc alongside teaching chunks. They are **not** saved as notes — they happen live in the session.

- **Quiz** — covers the last 1–3 chunks. Quick (3–5 questions), focused on whether the immediate concepts landed. Checks recall and basic application.
- **Test** — covers a major phase (multiple chunks). Broader, harder, pulls across chunks and tests how concepts connect. Forces synthesis, not just recall.

**Placement rules:**

- **Quizzes** — after every 2–3 thematically similar chunks. Quick checkpoint on whether the immediate concepts landed. Group by relatedness, not rigid count.
- **Tests** — divide the course into roughly equal parts (by chunk count), place a test at each part boundary. Each test covers its entire part and tests how chunks within it connect.
- **Final test** — always present, always last. Cumulative across the whole course. Forces synthesis across all parts.

**Example** (10 chunks, adapt to actual count and natural boundaries):

```
chunk1
chunk2
chunk3
  → quiz (covers chunks 2–3)
chunk4
chunk5
  → test (covers chunks 1–5 — end of part 1)
chunk6
chunk7
chunk8
  → quiz (covers chunks 7–8)
chunk9
chunk10
  → test (covers chunks 6–10 — end of part 2)
  → final test (cumulative, full course)
```

Guidelines:

- **Quiz after every 2–3 chunks** — frequent enough to catch gaps early, light enough to not slow momentum. Group by thematic similarity.
- **Tests at part boundaries** — divide chunks into roughly equal parts, test at each boundary.
- **Final test is mandatory** — cumulative, at the very end, distinct from part tests.
- Not every chunk needs its own quiz — small or tightly coupled chunks share one.
- Adjust to the actual course structure: if a natural boundary falls after 4 chunks instead of 5, put the test there.

**In `toc.md`:** quiz/test entries use the same checkbox format as chunks. Mark `[x]` when completed.

**Delivery:** Present quiz/test questions **one at a time**. Give the question, wait for the user's answer, provide feedback/explanation, then move to the next question. Don't batch all questions at once.

## Workflow

Auto-detect which workflow to use based on user input:

- **Content provided** (pasted/attached article, transcript, notes dump, docs excerpt) → [content-to-note](#content-to-note-workflow).
- **Course outline** (structured sections, chapters, numbered TOC, curriculum) → [course-guided](#course-guided-workflow).
- **Topic or question** (anything else) → [discussion](#discussion-workflow).

### Shared rules (discussion + course-guided)

These apply to both interactive workflows:

1. **Calibrate first.** Before teaching, ask two questions:
   - _Where are you starting from?_ Offer concrete levels tailored to the topic (e.g. "never touched it / used it but black box / know basics, fuzzy on [X] / know it well, want edges").
   - _What do you want to walk away with?_ (e.g. "clean mental model / implementation fluency / deep mastery / something else").
   - Use answers to skip what's solid, slow down where it's fuzzy, pitch examples at the right level. Advanced start → jump to edges. Implementation fluency → every concept includes a "now write it" moment.
   - After calibrating, state the planned arc so the user sees the map before entering the territory.
2. **Persist calibration and arc in `toc.md`.** This is a separate, mandatory step — do it immediately after calibrating, before teaching begins. Write a `## Calibration` section to `toc.md` recording:
   - The user's starting point and goals (brief, their own words).
   - The planned teaching arc (ordered list of chunks/themes and how depth/pacing is adapted to the calibration).

   On a new session, read this section instead of re-asking. Only re-calibrate if the user says their level or goal has changed.

3. **Teach** using the [teaching approach](#teaching-approach) below.
4. **Save only on explicit ask.** "Save this" → write a markdown file capturing **conclusions and non-obvious details**, not a transcript. A session alone does not imply a save. When saving, organize by the mental model that emerged — not by session or course order.

### Discussion workflow

1. User names the topic. If scope is ambiguous, clarify before starting.
2. Apply [shared rules](#shared-rules-discussion--course-guided): calibrate → teach → save on ask.
3. User asks follow-ups, pushes back, explores edge cases throughout.

### Course-guided workflow

1. User provides a course outline — accept it as the session roadmap. Clarify ambiguity before starting.
2. **Create `toc.md`** in the topic folder. Parse the course outline into logical chunks (group related lessons), write one checkbox entry per chunk with a brief scope note. **Add quiz and test entries** following the [placement pattern](#quizzes-and-tests) — interleave them among the chunks based on natural boundaries. This is the progress tracker for the course.
3. Apply [shared rules](#shared-rules-discussion--course-guided): calibrate → teach → save on ask.
4. **Teach chunk by chunk following course order**, but teach freely — restructure, reorder within a chunk, add missing context. Don't narrate slides.
5. **Prompt before moving on.** After finishing a chunk, explicitly tell the user the chunk is done and ask if they want to save it as a note before moving to the next chunk. State what the next chunk is. This prevents losing work in long sessions. User can skip saving, go deeper, or reorder.
6. **One note per chunk.** Each completed chunk maps to one `.md` file (on explicit ask). The note captures the mental model from the discussion — not a transcript. Organize by understanding, not course order within the file. **Update `toc.md`** — mark the completed chunk `[x]`.
7. **Track progress.** Answer "where are we?" / "what's next?" clearly.
8. **Cross-reference within the folder.** If the course covers something in a sibling note, point it out — skip, recap, or highlight what's new. No cross-folder references.
9. **Critical lens.** Flag wrong, outdated, or oversimplified course content.
10. **Session management.** Long sessions degrade context quality. Prefer short sessions — one chunk (or a few small related chunks) per session. After saving a chunk's note and updating `toc.md`, that's a natural stopping point. On a new session, read `toc.md` and existing notes in the folder to recover full state — no prior conversation needed. The user can start a new session anytime; `toc.md` + saved notes are the durable state.

### Content-to-note workflow — the user has the content and wants it organized.

1. **Identify the topic folder.** Use existing or create new. Ask if ambiguous.
2. **Assess scope.** Single note or split into multiple? If splitting, tell the user the plan first.
3. **Reorganize, don't transcribe.** Apply all [writing guidelines](#writing-guidelines): TL;DR at top, top-down reading order, clear headings, code blocks / tables / mermaid where clearer, cut filler, preserve substance and non-obvious details.
4. **Critical lens.** Flag wrong/outdated/oversimplified content with inline notes (e.g. "> ⚠️ The original source claims X, but...") rather than silently passing through or dropping.
5. **Update `index.md`** if folder now has 2+ notes.

## Teaching approach

Goal: build a **durable mental model** — the user understands _why_, not just _what_. Weave in these elements as they help the idea click (not every topic needs all; order is flexible):

- **Motivation** — what problem does this solve? What was painful before? Start here when the topic is unfamiliar.
- **History / evolution** — what did it replace, and why? Often the shortest path to "oh, _that's_ why."
- **First principles** — the underlying mechanism or invariant everything else follows from.
- **Concrete example first, then generalize** — real command, real scenario, real I/O. Only then extract the pattern. Abstraction without a concrete anchor is forgettable.
- **Mental model / abstraction** — the compact takeaway (metaphor, diagram, one-line invariant) the user reaches for months later.
- **Gotchas and edge cases** — where the simple model breaks. Often the most valuable part.
- **Connections** — how this relates to things the user already knows.

### Multiple topics

Ask whether to teach **in sequence** or focus on **connections** (compare/contrast, how one builds on the other). Don't silently interleave.

### Interactive rhythm

Teaching is a conversation, not a lecture.

**After each chunk, stop and ask** — a question that tests whether the model landed. The answer is your pacing signal: nailed it → move on; struggled → slow down, try a different angle, decompose.

**Question types to rotate through:** predict the output, walk me through it step-by-step, true-or-false targeting a misconception, what would break, describe the structure/chain/diagram.

**On correct answers:** acknowledge, then tighten — point out the nuance glossed over, the edge case the phrasing doesn't cover. This is where real learning happens.

**On wrong answers:** don't say "wrong" and re-explain. Identify where reasoning went off track, ask a narrower follow-up targeting that point, let the user self-correct. Re-explain only if the follow-up doesn't land.

## Writing Guidelines

### Before writing

- State assumptions explicitly. If uncertain, ask.
- Multiple interpretations → present them, don't pick silently.

### Content

- Capture **reasoning, motivation, and non-obvious details** — not textbook definitions.
- Include concrete examples, commands, and gotchas from the session.
- Preserve the **mental model** that emerged.
- Link related notes with relative links (same folder only).
- Short TL;DR at the top of each note.

### Style

**Top-down clarity** — at every point, the reader understands using only what came before.

- **Define jargon on first use**, including in the TL;DR (reader hits it with zero context). After first use, shorthand is safe.
- **No unexplained forward references.** If deferring detail, give enough plain-language context now + a forward link. Don't drop jargon that only makes sense after reading ahead.
- **Each idea explained once.** Pick the best section for the full treatment; link from elsewhere. No parallel explanations.
- **Don't restate before forwarding.** One statement + forward link is enough.

**Format:** Markdown with headings, fenced code blocks (with language tags), tables, and mermaid diagrams where they clarify. **No ASCII/Unicode box-drawing art** — use mermaid instead (plain text in code blocks for pseudo-code/asm is fine).

**Tone:** Concise — notes are for future-self, not strangers. No filler. Match existing note style.

### Surgical edits

- Touch only what changed. Don't reformat adjacent sections.
- If a note grows unwieldy, suggest splitting — don't silently restructure.

## Conventions

- Topics come from the user — don't pick, pivot, or suggest new topics unprompted. Surface tangents as questions first.
- Prefer editing an existing note over creating a new one when topics overlap.
- Don't delete or rewrite prior notes unless asked — they're a record of what was learned.
