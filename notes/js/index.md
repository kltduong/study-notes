# JavaScript — Notes

Standalone reference notes on JS mechanics that don't belong to a structured course. Focus is on mental-model clarifications and cross-language comparisons.

## Reading order

Order doesn't matter — each note is self-contained.

## Subtopic map

- [er-confusing.md](er-confusing.md) — Disambiguating "how many ERs exist?" (1:1 rule) from "how does the EC find them?" (two pointers).
- [this-self.md](this-self.md) — JS `this` vs Python `self`: two designs for the same problem, tradeoffs, determination rules by function kind, and the detachment footgun.

## Cross-cutting concepts

Both notes deal with **execution context internals** — the ER note covers the environment-record side, while the `this` note covers `[[ThisValue]]` on Function ERs. Together they map the two things an EC carries: variable bindings and a receiver.
