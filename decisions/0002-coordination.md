# ADR-2: Coordination

## Status

Accepted

## Record Completeness

Complete

## Context

Many specialized processes must act without a central conductor and without
direct agent-to-agent messaging (which propagates uncontrollably — see
bibliography §5). One coordination pattern is not enough: role collisions,
duplicate instances, and once-only jobs are three distinct problems.

## Decision

Coordinate by **stigmergy**: workers leave typed traces in the shared graph and
react to the graph, not to each other (the blackboard pattern). Who-acts-when is
governed by **three layers used together**:

- **(A) Worker-type scope** — a worker only acts within its type's partition,
  removing role collisions.
- **(B) Fenced claim + graph lease** for ordinary tasks — removes duplicate
  instances without a central boss. Lease renewal is **CAS from old `lease_until`
  to new `lease_until`**; lease duration is much larger than worker p99 latency.
  Contention uses **randomized backoff** and **per-worker task-affinity hashing**.
- **(C) Leader election** only for singleton work (sleep/rebuild/consolidation
  jobs that must run exactly once).

**Known edge:** singleton leadership is a scoped SPOF. Mitigated by fencing the
leader lease, making singleton work idempotent or fenced, and firing a **liveness
alarm if consolidation has not run by time `T`**.

## Consequences

- No combinatorial message fan-out; the graph is the single coordination surface.
- Coordination correctness depends on lease/lock discipline (ADR-1) and on loop
  stability (ADR-9).
- The graph lease removes duplicate workers without a central scheduler; leader
  election guarantees once-only jobs run once — the two cover different failure
  modes and are not interchangeable.

## Alternatives

- **Direct multi-agent communication** — rejected; uncontrolled propagation
  [Project Sid 2024; ChatDev 2023].
- **Pure ant-colony selection** for dozens of agents — rejected; too slow/noisy
  to converge at this scale.
- **A central orchestrator** — rejected; reintroduces the single conductor the
  architecture exists to avoid.
- **A global leader for all work** — rejected; makes every task depend on one
  process. Leader election is reserved for genuinely singleton jobs only.
