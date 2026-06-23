# ADR-4: Reward Source

## Status

Accepted

## Record Completeness

Complete

## Context

Learning is built in from day one, but a system that grades its own work drifts
(see reconsolidation / model-collapse, glossary §3, §5). The scope is a powerful
junior-level tool, not AGI; self-generated ground truth without external signal
is an open scientific problem, so the system avoids it and uses cheap objective
truth. Reward must come from outside the system.

## Decision

Reward is sourced **primarily from objective action outcome** — compile / test /
lint — plus **user correction as a first-class `user_correction` event**.
Self-critique is **auxiliary**, never the main signal. Schema slots for all reward
sources exist from day one. This is the same rule the dev loop runs on
(`../standards/verification.md`).

**Reference design — gated promotion loop:** conservative candidate filter →
empirical regression gate → reversible, attributed write → circularity guard.

**Required enforcement:**

- The **circularity guard is mechanical**, not advisory.
- Candidate filters **may not count correlated signals as independent axes**.
- **User silence is not approval.**
- The provenance check **rejects candidates whose confirmation traces back to
  previous `origin:"learned"` writes** — closing the self-confirmation loop.

## Consequences

- No self-reinforcing confidence loops from internal agreement alone.
- Learning is gated on availability of external signal; absence of signal means no
  reward, not a guessed one.
- Every promotion is reversible and attributed, so a bad learned write can be
  rolled back and traced.

## Alternatives

- **Self-grading / internal critic as reward** — rejected; drives drift and
  collapse.
- **Counting correlated signals as independent confirmation** — rejected; inflates
  confidence from what is really one source (the same independence hazard as ADR-3
  / ADR-9).
- **Treating user silence as approval** — rejected; absence of correction is not a
  reward signal.
