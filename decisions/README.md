# Decisions (ADRs)

Architecture Decision Records for **cross-repo** choices — decisions that bind
the whole `wefts` workspace, not one repo. Repo-local decisions live in the repo
that owns them (`swarm/docs/decisions/`, `hive/`), per
`../standards/how-to-write-adr.md`.

These are **locked** decisions. They are changed by *superseding* with a new ADR,
never by silently editing an accepted one.

Decision status and record completeness are separate. `Accepted` means the
decision is binding. `Stub` means this markdown record still needs the full
wording transplanted from the older spec.

## Index

- [ADR-0: Storage Engine](0000-storage-engine.md) — Accepted · Complete
- [ADR-1: Concurrency](0001-concurrency.md) — Accepted · Stub
- [ADR-2: Coordination](0002-coordination.md) — Accepted · Stub
- [ADR-3: Confidence Calculus](0003-confidence-calculus.md) — Accepted · Complete
- [ADR-4: Reward Source](0004-reward.md) — Accepted · Stub
- [ADR-5: Topology & Visibility](0005-topology-visibility.md) — Accepted · Stub
- [ADR-6: Embeddings](0006-embeddings.md) — Accepted · Stub
- [ADR-7: LLM I/O](0007-llm-io.md) — Accepted · Stub
- [ADR-8: Gate Thresholds](0008-thresholds.md) — Accepted · Stub
- [ADR-9: Stigmergic-Loop Stability](0009-stigmergic-loop-stability.md) —
  Accepted · Stub
- [ADR-10: `wefts` Workspace Split](0010-wefts-workspace-split.md) —
  Accepted · Complete
- [ADR-11: Hive Plugin Ownership](0011-hive-plugin-ownership.md) —
  Accepted · Complete
- [ADR-12: Operator Sync Boundary](0012-operator-sync-boundary.md) —
  Accepted · Complete

> Stub records are intentionally usable as anchors. They are not drafts of the
> decision; they are incomplete write-ups of already accepted decisions.
