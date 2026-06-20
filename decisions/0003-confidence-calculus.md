# ADR-3: Confidence Calculus

## Status

Accepted

## Record Completeness

Complete

## Context

Traversal reaches conclusions through chains and corroborating paths of partially
reliable edges. We need one coherent way to combine reliabilities along a path and
across paths, without double-counting shared evidence.

## Decision

- **AND (along a path):** product of edge reliabilities, computed in log-space.
- **OR (across independent paths):** noisy-OR.
- **Within a shared-ancestor group:** take the **max** (one source votes once).
- Across independent groups: noisy-OR of the per-group maxima.

Product and noisy-OR are De Morgan duals in one probabilistic algebra. This
**replaces** the earlier `min + noisy-OR` mix, which combined two different
algebras (a Zadeh t-norm with a probabilistic co-norm).

Full derivation, worked example, and open problem:
[../architecture/confidence-calculus.md](../architecture/confidence-calculus.md).

## Consequences

- Chaining and corroboration compose coherently regardless of graph shape.
- Correctness depends on detecting path independence — cheap detection at scale
  is an **open problem** (region-based / loop-corrected BP is the principled
  solution space).

## Alternatives

- **min + noisy-OR** — superseded; mixed algebras, incoherent composition.
- **Naive noisy-OR over all paths** — rejected; double-counts shared ancestors and
  manufactures overconfidence.
