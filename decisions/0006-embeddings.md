# ADR-6: Embeddings

## Status

Accepted

## Record Completeness

Complete

## Context

Embedding and inference are Python-ecosystem work; they should not live in the
Elixir kernel. Vectors are stored in the pgvector substrate (ADR-0). The hazard
this ADR closes: vectors from *different* embedding models share a vector space
only by accident — mixing them produces meaningless cosine scores, silently.

## Decision

**Every vector carries the namespace stamp of the model that produced it.**

- **Changing the embedding model forces a full re-embed**, run as a **singleton
  under leader election** (ADR-2) — separate from schema migration. The stamp is
  **not marked complete until the run has covered the whole corpus** — a
  self-healing migration, so an interrupted re-embed is detectable and resumable
  rather than leaving a silently mixed corpus.
- **Cosine thresholds are a function of the embedding model** and must be
  re-derived whenever it changes (ties to ADR-8's empirical bands).

Mechanically, embeddings are produced by a **Python sidecar over gRPC** and stored
via the storage port's vector axis (pgvector); the kernel never holds the ML
stack.

## Consequences

- Mixing vectors from different models is **structurally impossible**, not merely
  discouraged — the stamp makes a cross-model cosine query fail rather than return
  a garbage score.
- An interrupted migration self-heals: the incomplete stamp marks the corpus as
  not-yet-uniform until the re-embed finishes.
- An embed-model swap is expensive by design (full re-embed + threshold
  re-derivation, ADR-8) — the cost is the price of comparable vectors.
- Polyglot boundary at gRPC; the kernel stays free of the Python ML stack.
  Embedding choice is swappable behind the model port (ADR-7 / ports.md).

## Alternatives

- **No stamp / an implicit single-model assumption.** Rejected — the day the model
  changes, old and new vectors coexist and every cosine score silently degrades.
- **Re-embed without completion tracking.** Rejected — an interrupted migration
  leaves a mixed corpus with no signal that it is mixed.
- **A shared "universal" threshold across models.** Rejected — cosine geometry is
  model-specific; one threshold is wrong for all but one model.
