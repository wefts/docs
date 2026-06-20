# ADR-9: Stigmergic-Loop Stability

## Status

Accepted

## Record Completeness

Stub

## Context

Stigmergic feedback (workers reacting to traces they and others leave) can
reinforce itself into divergence or premature convergence. Formal convergence —
not just rate limits — is a named open problem.

## Decision

> TODO: transplant the current stability mechanism from
> swarm_architecture_spec.md (what is in place today — rate limits, decay,
> damping — and what guarantees it does and does not give).

## Consequences

- Current stability rests on rate limiting; **formal convergence is unproven** and
  remains open. Interacts with decay-vs-reinforcement tuning (lambda) and with
  cortical-voting-style premature convergence (glossary §4).
- **Reinforcement treats provenance-distinct events as independent — they are
  not.** `seen_count` increments once per distinct `(edge, provenance)` event
  (the `edge_provenance` guard in `Swarm.Graph.Store`). That guard is exact for
  one hazard — a *single* event must not count twice, which closes the endogenous
  confirmation loop — but silent on another: *distinct yet correlated* events. N
  derivatives of one source reinforce an edge N times, as if N independent
  witnesses had confirmed it. This is the same shared-ancestor overcounting that
  ADR-3 corrects in the **confidence** dimension, reappearing here in the
  **strength** dimension, where it is today only *mitigated* (Hill saturation
  bounds it; decay dominates a one-off burst), not *corrected*. Residual hazard:
  a connector emitting provenance-fresh but evidentially-stale events refreshes
  `last_seen` indefinitely and pins an edge near saturation, defeating decay —
  turning a decay-dominated edge effectively immortal with no new evidence behind
  it. Full analysis and solution space:
  [confidence-calculus.md → Independence in the strength dimension](../architecture/confidence-calculus.md#independence-in-the-strength-dimension).
  Open until a decision is taken among: connector provenance contract /
  per-source reinforcement cap / lineage-aware reinforcement.

## Alternatives

> TODO: transplant rejected stabilization approaches.
