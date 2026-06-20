# ADR-4: Reward Source

## Status

Accepted

## Context

Learning is built in from day one, but a system that grades its own work drifts
(see reconsolidation / model-collapse, glossary §3, §5). Reward must come from
outside the system.

## Decision

Reward is sourced **only from external ground truth** — e.g. tests pass, a build
succeeds, a user correction lands. The system **never grades itself**. This is the
same rule the dev loop runs on (`../standards/verification.md`).

> TODO: transplant the concrete reward signals and how they attach to graph edges
> from swarm_architecture_spec.md.

## Consequences

- No self-reinforcing confidence loops from internal agreement alone.
- Learning is gated on availability of external signal; absence of signal means no
  reward, not a guessed one.

## Alternatives

- **Self-grading / internal critic as reward** — rejected; drives drift and collapse.
