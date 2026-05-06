---
inclusion: always
---

# Learner Profile

Cross-course picture of the learner's background, preferences, and aggregated competence. Read at session start, alongside `courses/<topic>/toc.md` when one exists. Update only at end-of-course solidification — never per-chunk or per-session (that noise belongs in `toc.md`).

## Scope rule (hard)

Only folders under `courses/` with a workflow-produced `toc.md` feed the competence table. Content under `notes/` is reference material for later reading — it has no calibration or assessment history and never appears in the competence table.

> ⚙️ **Bootstrap note.** Items marked `🟡 inferred` came from existing course notes before a calibration happened. Confirm or correct them during the next calibration and upgrade to `✅ confirmed` once verified.

## Background

✅ confirmed (from `courses/async-js/` completion, `courses/js-inheritance/` in progress):

- Working JavaScript developer; comfortable with modern JS (ES6+, async/await, modules).
- Interest goes past day-to-day usage into spec-level mechanics (event loop, microtasks, `[[Prototype]]`).
- Not a beginner in the JS topics covered through the workflow so far.

## Learning Preferences

- **First principles over cookbook.** Wants _why_ before _what_. Definitional/procedural content alone lands poorly — derive behavior from fundamentals.
- **Formal/abstract structure when it exists.** If a behavior maps to a known mathematical or logical structure (quantifiers, algebraic identities, folds, type relationships), surface it explicitly. The formal framing is valued as a _separate layer of understanding_ — not a substitute for the mechanism, but a complement that reveals why the design is the only coherent choice. Derive behavior from the abstraction, don't just label it.
- **Mental model orientation.** Compact takeaways that survive months later beat exhaustive reference coverage.
- **Concrete → general.** Anchor the mechanism in a real example before abstracting.
- **Mechanism-level precision.** "Close but not quite" is unacceptable — give the precise version even when the conclusion was right. Refinement tracking exists for this.
- **Mermaid diagrams and comparison tables** land well. No ASCII/Unicode box-drawing art.
- **History/evolution** only when load-bearing (what it replaced and why). Skip pure chronology.
- **Rabbit holes flagged, not interleaved.** Sidenote / "for later" blocks work well.
- **Pace:** moderate on scheduling and mechanics; lighter on review of known material. Comfortable going deep when the _why_ requires it.

## Topic Competence (Global)

Absolute picture of what the learner currently knows per topic. Updated at end-of-course solidification. Level reflects actual understanding — not relative to any single course's goal.

| Topic                     | Level   | Notes                                                                                                                                                                                                         |
| ------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Async JS                  | `solid` | Event loop, promises, microtasks, async/await, combinators, testing, rAF — all drilled and tested. Strong on execution-order reasoning; minor pattern of right-conclusion-imprecise-mechanism under pressure. |
| JS prototypal inheritance | `shaky` | Core mechanics (chain walk, `[[Prototype]]`, instantiation patterns, `.prototype` property) solid. Course ~70% done — building chains, OOP/class, composition not yet covered.                                |

**Not tracked here:** `notes/` content. Notes are reference material for later reading — they have no calibration or assessment history and do not feed this table.

## Cross-Cutting Strengths

Anchor points to reach for when teaching adjacent topics:

- **State machines + pointer/offset thinking.** Promises-as-state-machine, shape pointers, `[[Prototype]]` links all landed cleanly.
- **Queue and scheduling mental models.** Task queue, microtask queue, FIFO ordering, drain semantics all reasoned well.
- **Derivation from two fundamentals.** Given a compact pair of invariants, derives consequences correctly (e.g. promise state-machine + `.then()` scheduling rule).
- **Engine/runtime split.** The "single-threaded engine, multi-threaded runtime" distinction is internalized.
- **Formal abstraction recognition.** Spots when a system's behavior maps to known math/logic structures (∀/∃ quantifiers, fold identities, vacuous truth) and uses the formal layer to reason about edge cases (e.g. empty-input combinator behavior).

## Cross-Cutting Weaknesses / Refinement Patterns

Recurring imprecisions pulled from `🔧 Refinements` across `courses/*/toc.md`. Probe proactively on related topics rather than waiting for them to resurface.

- **Engine vs runtime conflation.** "The engine places the callback in the queue" (it's the runtime). Resurfaces under pressure.
- **Descriptor vs value confusion.** `{ value: 42 }` (descriptor) vs `42` (the value). Shows up with `defineProperty` and anywhere property access involves metadata.
- **"Internal slot" over-reach.** Tags regular JS objects (e.g. `Dog.prototype`) as "engine internals." The link is engine-managed; the target is an ordinary object.
- **Right conclusion, wrong mechanism.** Predicts the correct outcome via a timing/coincidence argument rather than the structural one (e.g. non-interleaving of click handlers "because clicks are slower").
- **Problem-category compression.** Merges related-but-distinct problems into fewer buckets (callback hell: inversion of control / composition / scattered errors → compressed to "nesting + readability").
- **Suspension model imprecision.** Describes `await` as "keeping a reference to a callback" rather than "suspends execution context and registers the continuation as a `.then()` handler on the awaited promise."

## Session Calibration Defaults

Default assumptions — shorten or skip the two calibration questions unless the topic falls outside the profile:

- **Starting point:** "knows basics, fuzzy on [specific subtopic]" — rarely "never touched it."
- **Goal:** clean mental model + implementation fluency. Deep mastery is welcome where relevant but not the default ask.
- **Arc:** motivation → first principles → concrete example → generalize → gotchas → connections. Skip history unless it explains a surviving design choice.
- **Confirm, don't re-ask.** State the assumed calibration in one line, invite correction, proceed if uncorrected.

## Maintenance

Update only on these triggers:

1. **End-of-course solidification.** Refresh that topic's competence row with the learner's current absolute level. Fold recurring refinement themes into the cross-cutting weaknesses list. Upgrade relevant `🟡 inferred` items to `✅ confirmed`.
2. **Cross-course refinement patterns.** If the same imprecision surfaces in two or more courses, add it to the weaknesses list.
3. **Explicit user correction.** If the user says "I'm stronger/weaker on X than you assumed," update immediately.
