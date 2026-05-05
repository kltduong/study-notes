---
inclusion: always
---

# Learner Profile

Cross-course picture of the learner's background, preferences, and aggregated competence. Competence is tracked only for courses under `courses/<topic>/` — content under `notes/` has no calibration or assessment history and is not tracked here. Read this alongside any `courses/<topic>/toc.md` at session start. Update at end-of-course solidification (not per-session).

> ⚙️ **Bootstrap note.** Sections below were inferred from existing notes. Confirm and correct in calibration — mark `🟡 inferred` items as `✅ confirmed` or edit.

## Background

🟡 inferred (from courses only — `courses/async-js/`, `courses/js-inheritance/`):

- Working JavaScript developer; comfortable with modern JS (ES6+, async/await, modules).
- Interest extends past day-to-day usage into spec-level mechanics (event loop, microtasks, `[[Prototype]]`).
- Not a total beginner in the JS topics taught through the workflow so far.

Content under `notes/` (e.g. `notes/git/`, `notes/css/`, `notes/client-side/`) is pre-existing or discussion-captured material without calibration history. Do not infer background from it.

## Learning Preferences

- **First principles over cookbook.** Wants _why_ before _what_. Definitional/procedural content alone lands poorly — notes should derive behavior from fundamentals.
- **Mental model orientation.** Values compact takeaways that survive months later over exhaustive reference coverage.
- **Concrete → general.** Prefers a real example anchoring the mechanism before abstraction.
- **Mechanism-level precision.** When an explanation is "close but not quite," wants the precise version — even if the conclusion was right. Refinement tracking reflects this.
- **Mermaid diagrams and comparison tables** land well. No ASCII/Unicode art.
- **History/evolution** only when load-bearing (what it replaced and why). Skip pure chronology.
- **Rabbit holes flagged, not interleaved.** Sidenote/"for later" blocks work well.
- **Pace:** moderate on scheduling/mechanics; lighter on review of known material. Comfortable going deep when the "why" needs it.

## Topic Competence (Aggregated)

Coarse rollup per course under `courses/`. Refresh at end-of-course solidification. Only folders under `courses/` (with a `toc.md`) appear here — content under `notes/` is not tracked.

Competence is **relative to the course's stated goal** in `toc.md` → `## Calibration` → _Goal_, not an absolute scale. `solid` on a "clean mental model" goal ≠ `solid` on a "deep mastery" goal. When reading a row, consult the course's goal for what the level means.

| Course                    | Goal (per `toc.md`)                                     | Level vs goal  | Notes                                                                                                                                                                                                                                                       |
| ------------------------- | ------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `courses/async-js/`       | Clean mental model of how async JS works under the hood | `solid`        | Event loop, promises, microtasks, async/await, combinators, testing — all drilled. Final test pending. Strong on execution-order reasoning.                                                                                                                 |
| `courses/js-inheritance/` | Mental model + implementation fluency + deep mastery    | `shaky` so far | Core mechanics (chain walk, `[[Prototype]]`, instantiation patterns) solid against a mental-model bar. Course ~70% done — building chains, OOP/class, composition remaining. Level against the full three-part goal can't be set until the course finishes. |

**Not tracked** (content under `notes/`, no course workflow history): `notes/git/`, `notes/css/`, `notes/client-side/`. If the learner later takes a course on any of these subjects, a new folder under `courses/` will be created with its own `toc.md` and added here.

## Cross-Cutting Strengths

Anchor points I can reach for when teaching adjacent topics:

- **State machines + pointer/offset thinking.** Promises-as-state-machine, shape pointers, `[[Prototype]]` links all landed cleanly.
- **Queue / scheduling mental models.** Task queue, microtask queue, FIFO ordering, drain semantics reasoned well.
- **Derivation from two fundamentals.** Given a compact pair of invariants, derives consequences correctly (e.g. promise state-machine + `.then()` scheduling rule).
- **Comfortable with engine/runtime split.** The "single-threaded engine, multi-threaded runtime" distinction is internalized.

## Cross-Cutting Weaknesses / Refinement Patterns

Recurring imprecisions **across courses under `courses/`** (only sources with `toc.md` history). Probe proactively on related topics. Sourced from `🔧 Refinements` in each course's `toc.md`.

- **Engine vs runtime conflation.** "The engine places the callback in the queue" (it's the runtime). Resurfaces under pressure.
- **Descriptor vs value confusion.** `{ value: 42 }` (descriptor) vs `42` (the value). Shows up with `defineProperty` and anywhere property access involves metadata.
- **"Internal slot" over-reach.** Tags regular JS objects (e.g. `Dog.prototype`) as "engine internals." The link is engine-managed; the target is ordinary.
- **Right conclusion, wrong mechanism.** Sometimes predicts the correct outcome but via a timing/coincidence argument rather than the structural one (e.g. non-interleaving of click handlers "because clicks are slower").
- **Three-vs-two problem compression.** Tends to merge related-but-distinct problems into fewer categories (callback hell: inversion of control / composition / scattered errors → got compressed to "nesting + readability").
- **Suspension model imprecision.** Describes `await` as "keeping a reference to a callback" rather than "suspends execution context + registers continuation as `.then()` handler on the awaited promise."

## Session Calibration Defaults

Default assumptions to shorten or skip the two calibration questions unless the topic is outside the profile:

- **Starting point:** "knows basics, fuzzy on [specific subtopic]" — rarely "never touched it."
- **Goal:** clean mental model + implementation fluency. Deep mastery is welcome where relevant but not the default ask.
- **Arc preference:** motivation → first principles → concrete example → generalize → gotchas → connections. Skip history unless it explains a surviving design choice.
- **Confirm, don't re-ask.** State the assumed calibration in one line, invite correction, proceed if none.

## Maintenance

Update triggers:

1. **End-of-course solidification pass** — refresh course competence, fold recurring refinement themes into the cross-cutting list, upgrade `🟡 inferred` → `✅ confirmed` where applicable.
2. **Cross-course refinement patterns** — if the same imprecision surfaces across two+ courses, add it to the weaknesses list.
3. **Explicit user correction** — if the user says "I'm stronger/weaker on X than you assumed," update immediately.

Do not update per-chunk or per-session — that noise belongs in `toc.md`. Never add rows for content under `notes/`.
