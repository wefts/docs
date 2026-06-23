# ADR-5: Topology & Visibility

## Status

Accepted

## Record Completeness

Complete

## Context

Who-can-see-what in the shared graph is **security-critical**: the visibility filter
is the privacy boundary, and it must hold under load. Topology (how nodes/edges are
grouped and scoped) and visibility are one decision: the choice to keep a *single*
graph rather than a graph-per-user is exactly what makes the filter load-bearing.

## Decision

**One graph, not a graph per user.** Privacy and context-dependent behavior are
enforced through a `visibility-scope` with **default-deny**.

- **Scope is materialized**, not computed per traversal. It lives as indexed labels
  / partitioned edge types so the filter prunes at the index, never as a per-edge
  traversal predicate evaluated at query time.
- **Single enforcement point, both endpoints.** A relation is traversable only if
  `edge.visibility_scope` is in the context's allowed-scope set, **and** a node is
  disclosable only if `node.scope` is in that set. Ingestion holds
  `edge.visibility_scope ≤ narrowest endpoint scope`, so the two always agree;
  enforcing both is fail-safe. An empty/absent allowed set discloses nothing (no
  query is even issued).
- **Derived-scope inheritance.** A node produced by consolidation from private
  instances **inherits the narrowest parent scope** by default. *Widening* scope
  requires min-support `k` from independent sources.

In the kernel this is `Swarm.Gate.Visibility` — the one place a context's allowed
scopes become a filter argument; `Swarm.Graph.Traverse` is the mechanism (index
pruning given `:scopes`) and decides no policy. Workers/connectors must route
visibility decisions through that module, never re-implement them.

## Consequences

- Gives **brain-like shared memory and situational retrieval** while preventing
  instance leakage and derived-knowledge leakage — the payoff that justifies the
  single-graph risk.
- The filter is **index-level pruning** (a performance property) rather than a
  query-time predicate (correctness-fragile and slow) — the two goals align here.
- The visibility-filter **threat model under load** remains a named open problem
  (OP#4): the *policy* is pinned and the enforcement point is single, but
  adversarial behavior at scale (timing, partial-result inference, scope-widening
  pressure) is not yet proven out. Privacy here is a security property, not a
  preference — addressed in T2 (graph-integrity contract).
- Scope-widening is deliberately expensive (needs `k` independent sources), which
  bounds how fast private knowledge can become shared.

## Alternatives

- **A graph per user / per instance.** Rejected — no shared memory, no
  cross-context consolidation; defeats the cognitive-swarm premise.
- **Per-edge traversal predicate computed at query time.** Rejected — slow at
  depth, and correctness-fragile (one missed predicate leaks); materialized
  indexed scope is both faster and fail-safe.
- **Allow-by-default with redaction.** Rejected — any gap leaks; default-deny
  makes the failure mode "too little visible," not "too much."
