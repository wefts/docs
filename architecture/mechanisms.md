# Reusable mechanisms

A catalog of borrowable building blocks for Swarm — substrate, runtime, scale, and
evolution. Each entry is a generic mechanism, then how it maps onto our kernel and
which port or decision it touches, then the caveat. These are candidates to adopt
and reconcile against the kernel, not settled canon; where the kernel already
implements one, treat this as the rationale, not a second source of truth.

The through-line: **durability and ordering belong in the Postgres substrate, not in
a transport**, and **the typed port is the seam that lets every layer evolve, scale,
and recover independently**. Keep both in mind below.

## Change propagation and ordering (substrate)

### Transactional outbox / CDC as the stigmergy signal

Workers don't message each other; they react to changes in the shared graph. The
mechanism that turns a write into a reaction: in the same transaction that mutates
the graph, append the change to an outbox (or rely on the write-ahead log directly),
then a single in-process reader tails it via logical replication and wakes the
workers that care. No broker, no separate service — change-data-capture over the
substrate we already have.

- *Maps to:* the Worker / stigmergy port; validates Postgres-as-substrate from the
  storage spike.
- *Caveat:* an outbox roughly doubles WAL volume; tailing the WAL directly is cheaper
  than an outbox table but couples you to the engine's replication format. Irrelevant
  at our scale — record it so a future scale-up doesn't relearn it.

### Monotonic sequence + gap detection

Order and integrity of the trace stream come from a plain monotonic sequence plus an
explicit gap check: if entries 1, 2, 4 land, then 3 is either in flight (wait and
re-read) or rolled back (skip after a timeout). It is a stripped-down logical clock
whose real payoff is the ability to *prove* nothing was dropped, not just to sort.

- *Maps to:* ordered consumption of traces; the feedback-loop-stability open problem
  — out-of-order or silently-dropped reinforcement is how a swarm drifts.
- *Caveat:* the gap-wait timeout is a real tuning knob; too short skips live events,
  too long stalls the lane.

### Idempotency at the boundary (exactly-once is a pipeline property)

No component delivers "exactly once" on its own — you engineer it across the pipeline:
every effectful action carries an idempotency key, the boundary dedups on that key,
and cursor/offset commits are transactional with the work they acknowledge. Retried
work then converges instead of doubling.

- *Maps to:* the Tool port (a retried outward action must not double-charge,
  double-send, double-write) and to re-processing traces after a crash. The kernel's
  provenance-keyed observation counting is already this pattern on the ingest side —
  extend the same discipline outward.
- *Caveat:* idempotency keys must be stable across retries and survive process death,
  i.e. derived from the work, not generated per attempt.

## Concurrency, distribution, and scale (runtime)

### Partition-by-key for ordered parallelism

The core scale primitive: partition the work stream by a key so that everything under
one key is processed in strict order on one lane, while different keys run fully in
parallel. Throughput scales with the number of keys; ordering is preserved exactly
where it matters and nowhere it doesn't. The *choice of key is the design decision* —
pick it at the consistency grain you actually need (per-node, per-subgraph,
per-entity), because too fine a key reorders related work and too coarse a key
serializes everything.

- *Maps to:* distributing reinforcement / linking work across many workers without
  racing on the same subgraph; the "smart system of dumb parts" only stays correct if
  same-target traces share a lane.
- *Caveat:* re-keying later means re-partitioning live state — choose deliberately up
  front.

### Demand-driven backpressure

A fast producer must not outrun slow consumers. Instead of a producer pushing until
queues overflow (and then flooding or silently dropping), consumers signal demand and
producers emit only what was asked for — bounded, demand-driven flow. In Elixir this
is the idiomatic `GenStage`/`Flow` shape, and it is the right backbone for the
observer→linker→classifier pipeline.

- *Maps to:* an observer that can flood the graph faster than linkers and classifiers
  drain it; backpressure keeps the swarm from thrashing under a burst.
- *Caveat:* backpressure makes the slowest stage the system's pace-setter — that's
  correct, but it means you scale the bottleneck stage explicitly, not the whole line.

### Supervision and let-it-crash

Workers don't defensively guard every error path; they crash on the unexpected and a
supervisor restarts them clean from known-good state. This is OTP's core mechanism and
already the kernel's ethos — the reuse note is to keep *transient* worker state
reconstructable from the graph, so a restart loses nothing that matters.

- *Maps to:* every worker behind the Worker port.
- *Caveat:* "let it crash" only stays cheap if restart is cheap — durable truth lives
  in the substrate, not in process memory.

### Leases and leader election with explicit quorum

Some work must run as a singleton — the CDC reader, a single owner over a hot
subgraph. The mechanism is a lease (time-bounded ownership) with leader election, and
the non-negotiable part is that **split-brain is handled on purpose**: OTP and BEAM
give you the primitives (distributed naming, `:global`, lease libraries) but not a
correct quorum for free. A runtime that "never goes down" still partitions, and a
partition with no quorum rule means two owners diverging.

- *Maps to:* the concurrency / coordination decision (leases, leader election).
- *Caveat:* local-first shrinks the blast radius, but the moment state replicates
  (substrate → remote node, or two kernel instances) you owe an explicit consistency
  model. Decide it; don't inherit it from the runtime's reputation.

### Scale the cheap path, gate the expensive one

Scalability here is asymmetric by design: cheap specialized workers can fan out
freely, but the expensive escalation (the large-model consilium) must stay
rate-limited and gated *regardless of load*. The reusable mechanism is backpressure
and admission control specifically on the escalation path, so a spike in cheap work
can never translate into a spike in expensive calls.

- *Maps to:* the gate thresholds decision and the cost-asymmetry premise of the whole
  system.
- *Caveat:* this means the gate needs a load-aware admission policy, not just a
  per-task difficulty score — cold-start thresholds are already an open problem here.

## Evolution without downtime

### Hot code update, with the typed port as the stability seam

A persistent, always-on assistant should improve without a cold restart — you want to
ship a better worker, classifier, or skill without losing the running graph or the
swarm's in-memory state. The unifying mechanism is that the **typed port is the seam**:
because adapters attach through a stable Protobuf contract, each tier can be updated
by a technique suited to its trust and runtime, while the kernel keeps running.

- **In-kernel Elixir (trusted):** true BEAM hot code loading. Two versions of a
  module live at once; processes migrate on their next fully-qualified call, and a
  `GenServer` transforms its state through `code_change/3`. Reserve real release-level
  hot upgrades (appup/relup) for the kernel, where state continuity is the point;
  elsewhere they are more ceremony than they're worth.
- **Out-of-process gRPC sidecars (heavy, polyglot — e.g. the embedding/inference
  sidecar):** no BEAM hot-swap, so update by blue-green at the port boundary — bring
  the new version up behind the same typed port, drain, cut over, retire the old. The
  Protobuf contract is what makes the swap invisible to the kernel.
- **WASM skills (untrusted community code):** swap by instantiating the new module and
  atomically switching the reference; the sandbox makes this the cheapest and safest
  tier to update live. Version the skill so a bad swap rolls back to the prior
  instance.

- *Maps to:* the plugin model (Connector / Worker / Channel / Model / Skill / Tool
  ports) and the three isolation tiers; directly serves the "persistent, local-first,
  never-rebooted" goal and continuous learning without amnesia.
- *Caveat:* a live code update is a deliberate operator action, never an agent default
  — same class of guarded operation as remote sync. State-shape changes still need a
  migration path (`code_change`, or a versioned port message), so treat schema-of-state
  evolution as part of the upgrade, not a separate afterthought.

## Reconcile against the kernel

Several of these already exist in some form — provenance-keyed observation counting
(idempotent ingest), supervision, the gate/consilium split. Before adopting an entry,
diff it against the running code; this catalog is the *why*, the kernel is the *what
is true now*.