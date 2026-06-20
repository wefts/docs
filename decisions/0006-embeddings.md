# ADR-6: Embeddings

## Status

Accepted

## Context

Embedding and inference are Python-ecosystem work; they should not live in the Elixir
kernel. Vectors are stored in the pgvector substrate (ADR-0).

## Decision

Embeddings are produced by a **Python sidecar over gRPC** and stored via the storage
port's vector axis (pgvector).

> TODO: transplant the specific embedding model, dimensionality, and update policy
> from swarm_architecture_spec.md.

## Consequences

- Polyglot boundary at gRPC; the kernel stays free of the Python ML stack.
- Embedding choice is swappable behind the model port (ADR-7 / ports.md).

## Alternatives

> TODO: transplant rejected embedding approaches.
