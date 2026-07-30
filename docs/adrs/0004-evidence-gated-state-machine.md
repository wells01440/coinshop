# 0004. Stage transitions are evidence-gated, gates are pure functions

Date: 2026-07-29 · Status: accepted

Context: time between workflow steps is unbounded; human memory is
not a state store; ancients require exploratory photography that
resists checklist thinking.

Decision: stages advance only through promote(); gate(to, facts) is a
pure function over a CoinFacts snapshot returning exactly what is
missing. State lives in the DB, never in anyone's head.

Consequences: unit-testable without a DB; failed promotes are
self-documenting; adding a gate is a one-function change. (R5)
