# ADR-0: Storage Engine

## Status

Accepted (settled by benchmark, not taste)

## Context

The storage substrate is the one load-bearing, hard-to-retrofit choice. The
graph needs CRUD, atomic compare-and-set (CAS), and vector similarity, plus
traversal over typed links. Candidates evaluated: **Postgres + pgvector** vs
**Memgraph**, on real hardware.

## Decision

Use **Postgres + pgvector**. It won or tied traversal at every scale tested and won
the hard-to-retrofit CAS axis by ~5×.

The storage **port** abstracts **CRUD + CAS + vector** only. Traversal queries are
engine-specific and are deliberately *not* behind the port — they would be rewritten
on any future migration.

## Consequences

- Single mature substrate for relational state, CAS, and vectors; one operational
  surface instead of two.
- Traversal is coupled to Postgres; a future engine change means rewriting traversal,
  by design and on record.

## Alternatives

- **Memgraph** — rejected. Caveat on record: its traversal was *not*
  BFS-optimized in the spike, so **re-test if traversal ever becomes the
  dominant workload.**
