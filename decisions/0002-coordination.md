# ADR-2: Coordination

## Status

Accepted

## Record Completeness

Stub

## Context

Many specialized processes must act without a central conductor and without
direct agent-to-agent messaging (which propagates uncontrollably — see
bibliography §5).

## Decision

Coordinate by **stigmergy**: workers leave typed traces in the shared graph and
react to the graph, not to each other (the blackboard pattern). Who-acts-when is
governed by **leases and leader election** in the Elixir kernel.

> TODO: transplant the concrete lease/leader-election rules and trace semantics
> from swarm_architecture_spec.md.

## Consequences

- No combinatorial message fan-out; the graph is the single coordination surface.
- Coordination correctness depends on lease/lock discipline (ADR-1) and on loop
  stability (ADR-9).

## Alternatives

- **Direct multi-agent communication** — rejected; uncontrolled propagation
  [Project Sid 2024; ChatDev 2023].
