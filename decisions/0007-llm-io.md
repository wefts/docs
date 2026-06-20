# ADR-7: LLM I/O

## Status

Accepted

## Context

Large models are rare, deliberate escalations (gate + consilium). Their input and
output need a stable, typed contract so disagreement can be captured rather than
flattened.

## Decision

> TODO: transplant the LLM input/output contract from swarm_architecture_spec.md
> (request/response shape, how model disagreement is preserved as a confidence signal,
> how the consilium aggregates).

## Consequences

- Model disagreement is kept as signal, not discarded (ties to ADR-3 and to
  `../standards/verification.md`).

## Alternatives

> TODO: transplant rejected I/O contracts.
