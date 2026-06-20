# Decisions (ADRs)

Architecture Decision Records for **cross-repo** choices — decisions that bind
the whole Wefts workspace, not one repo. Repo-local decisions live in the repo
that owns them (`swarm/docs/decisions/`, `hive/`), per
`../standards/how-to-write-adr.md`.

These are **locked** decisions. They are changed by *superseding* with a new ADR,
never by silently editing an accepted one.

## Index

- [ADR-0: Storage Engine](0000-storage-engine.md) — Accepted (settled by benchmark)
- [ADR-1: Concurrency](0001-concurrency.md) — Accepted
- [ADR-2: Coordination](0002-coordination.md) — Accepted
- [ADR-3: Confidence Calculus](0003-confidence-calculus.md) — Accepted
- [ADR-4: Reward Source](0004-reward.md) — Accepted
- [ADR-5: Topology & Visibility](0005-topology-visibility.md) — Accepted
- [ADR-6: Embeddings](0006-embeddings.md) — Accepted
- [ADR-7: LLM I/O](0007-llm-io.md) — Accepted
- [ADR-8: Gate Thresholds](0008-thresholds.md) — Accepted
- [ADR-9: Stigmergic-Loop Stability](0009-stigmergic-loop-stability.md) —
  Accepted
- [ADR-10: Wefts Workspace Split](0010-wefts-workspace-split.md) — Accepted
- [ADR-11: Hive Plugin Ownership](0011-hive-plugin-ownership.md) — Accepted
- [ADR-12: Operator Sync Boundary](0012-operator-sync-boundary.md) — Accepted

> Many records below are **skeletons**: title, status, and the parts derivable
> from the project summary are filled; the full decision text is marked
> `TODO: transplant from swarm_architecture_spec.md`. Transplant the real
> wording so these stop being stubs — the numbering and anchors are already
> correct.
