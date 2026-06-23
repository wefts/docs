# ADR-1: Concurrency

## Status

Accepted

## Record Completeness

Complete

## Context

Concurrency and process supervision are hard to retrofit. They must be a
property of the kernel from day one, not bolted on later. Many workers write the
shared graph in parallel, so races are inevitable; the kernel must make the
coordination invariants structural, not advisory.

## Decision

Logic and coordination run on **Elixir/OTP** (supervision trees, "let it crash"),
with concurrency invariants baked into the kernel skeleton from the first build
slice:

- **Mutations go through engine transactions.**
- **Coordination/claims use compare-and-swap (CAS)** on node properties with a
  **monotonic fencing token** — closing the slow-but-alive holder problem (a
  stalled worker's late write is rejected because its token is stale).
- **Mixed consistency, deliberately.** Strong consistency is *required* for
  claim/lease, irreversible actions, and graph-region read-modify-write such as
  consolidation. Eventual consistency is *allowed only* for independent derived
  data. Consolidation is RMW over a large region; under eventual consistency it
  clobbers parallel ingest.
- **Writes are idempotent.** The natural key is
  `(source, type, target, visibility-scope)`; upsert means insert-or-increment,
  and `seen_count` increments are atomic.

## Consequences

- Supervision, restarts, and back-pressure come from a mature runtime.
- The kernel's hard core is Elixir; polyglot work lives outside it (see ADR-6,
  ADR-7).
- Fencing tokens make a stalled-then-revived worker safe by construction, not by
  hope.
- The idempotent natural key is what lets reinforcement (ADR-9) and confidence
  (ADR-3) treat a re-observation as an increment rather than a duplicate node.

## Alternatives

- **Leases without fencing.** Rejected — a slow-but-alive holder resumes and
  writes after its lease lapsed, corrupting state a successor already owns.
- **A global lock over the whole graph.** Rejected — serializes all writers and
  destroys the parallelism that motivates the swarm; locality (per-key CAS +
  leases) is the point.
