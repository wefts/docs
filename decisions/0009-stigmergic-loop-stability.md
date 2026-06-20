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

## Alternatives

> TODO: transplant rejected stabilization approaches.
