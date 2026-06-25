# Decisions (ADRs)

Architecture Decision Records for **cross-repo** choices — decisions that bind
the whole `wefts` workspace, not one repo. Repo-local decisions live in the repo
that owns them (`swarm/docs/decisions/`, `hive/`), per
`../standards/how-to-write-adr.md`.

These are **locked** decisions. They are changed by *superseding* with a new ADR,
never by silently editing an accepted one.

Decision status and record completeness are separate. `Accepted` means the
decision is binding; `Complete` means the markdown record carries the full
wording (a `Stub` record would be an accepted decision whose write-up still
needs transplanting). **All records below are now Complete.**

## Index

- [ADR-0: Storage Engine](0000-storage-engine.md) — Accepted · Complete
- [ADR-1: Concurrency](0001-concurrency.md) — Accepted · Complete
- [ADR-2: Coordination](0002-coordination.md) — Accepted · Complete
- [ADR-3: Confidence Calculus](0003-confidence-calculus.md) — Accepted · Complete
- [ADR-4: Reward Source](0004-reward.md) — Accepted · Complete
- [ADR-5: Topology & Visibility](0005-topology-visibility.md) — Accepted · Complete
- [ADR-6: Embeddings](0006-embeddings.md) — Accepted · Complete
- [ADR-7: LLM I/O](0007-llm-io.md) — Accepted · Complete
- [ADR-8: Gate Thresholds](0008-thresholds.md) — Accepted · Complete
- [ADR-9: Stigmergic-Loop Stability](0009-stigmergic-loop-stability.md) —
  Accepted · Complete
- [ADR-10: `wefts` Workspace Split](0010-wefts-workspace-split.md) —
  Accepted · Complete
- [ADR-11: Hive Plugin Ownership](0011-hive-plugin-ownership.md) —
  Accepted · Complete
- [ADR-12: Operator Sync Boundary](0012-operator-sync-boundary.md) —
  Accepted · Complete
- [ADR-13: Evidential Origin](0013-evidential-origin.md) —
  Accepted · Complete

> Each record is the full write-up of an already-accepted decision; the
> decisions themselves are locked and changed only by a superseding ADR.
