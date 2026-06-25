# ADR-13: Evidential Origin

## Status

Proposed

## Record Completeness

Complete

## Context

The independence trap — *correlated evidence must not be counted as independent
corroboration* — is the named open problem of this calculus. It has two faces,
both still open in the read path:

- **Confidence dimension (ADR-3).** `Swarm.Graph.Confidence.combine_typed/1`
  exists and is correct (LLM-generated kinds collapse to one shared-ancestor
  group; external observations are independent). But it has **no production
  caller** — `Swarm.Graph.Traverse` computes a naive chain product
  `w.conf · e.reliability · decay(age)` with **no lineage grouping at all**. The
  defense is written and unwired.
- **Strength dimension (ADR-9).** `seen_count` increments once per distinct
  `(edge, provenance)` event. `provenance` today is an *emission instance*, not
  an *evidential origin*. N derivatives of one upstream source carry N distinct
  provenance keys and reinforce an edge N times, as if N independent witnesses
  confirmed it — the immortal-edge hazard named in
  [`confidence-calculus.md` → Independence in the strength dimension](../architecture/confidence-calculus.md#independence-in-the-strength-dimension).

These were deferred because nothing generated multi-origin claims, so neither
path was hot. The **cognitive-activation spike** (2026-06-25, disposable epoch on
`swarm_slice`; `board/research/cognitive-activation-spike.md`) changed that: it
forced enrichment to fire (7.9 useful S-P-O triples/node; 150 claims / 239
entities / 148 capped claim edges) and ground-truthed the accounting layer as
**dead code on the path that will carry the cascade once a real enrichment worker
ships**. It also measured the *shape* of the hazard:

- Exact-(s,p,o) corroboration was **0/148** — every claim edge single-source.
  Over-corroboration is therefore **semantic, not lexical**: it lives in the
  **24 near-duplicate entity pairs** a dense reader found and exact keys missed.
- So origin accounting is **coupled to entity resolution**: today fragmentation
  *hides* corroboration (two spellings never combine); a naive dense soft-merge
  of those pairs *before* origin accounting would **manufacture** the very
  correlated-evidence inflation this decision exists to prevent.

A 2-family council (gemini-3.1-pro + local gemma4, both SOUND-WITH-CAVEATS;
journal 2026-06-25) added: (1) a budget breaker is an emergency fuse, not a
scheduler — the missing piece is a reward-gate/priority control plane that shares
this same lineage primitive, so the two must be **co-designed**; (2)
origin≠derivative separation is partly a *semantic* judgment with real compute
cost — it must be budgeted, not assumed structural and free.

This is a **cross-repo correctness invariant**: it binds the kernel read path
(swarm), any future enrichment worker (swarm), and the connector provenance
contract at the ingest boundary (hive plugins / `ports.md`). Hence a workspace
ADR, decided once, implemented in `swarm/`.

## Decision

Make **evidential origin a first-class property of the graph**, and wire the
existing independence defenses onto it. Four parts, one shared lineage primitive:

1. **Origin is first-class and distinct from emission instance.** Every
   reinforcement carries an **origin key** (evidential source identity) separate
   from the per-event provenance key (emission instance). The connector
   provenance contract (`ports.md`, enforced at ingest) requires the origin key
   to be derived from *source/content identity*: re-emitting the same fact reuses
   the same origin, a genuinely independent source gets a new one. This is the
   cheapest correct cut and pushes origin determination to the boundary where it
   is actually known.

2. **Reinforcement and corroboration count distinct origins, not distinct
   events.** `seen_count` (and any future confidence consumer of it) is computed
   over **distinct origins**, with a **per-origin reinforcement ceiling** — the
   strength-dimension mirror of ADR-3's "max within a shared-ancestor group". One
   origin cannot reinforce past the ceiling no matter how many derivative events
   it emits, closing the immortal-edge hazard.

3. **Wire the typed grouping into the read path.** `combine_typed/1` (or its
   successor) is called wherever cross-origin corroboration is computed, so that
   co-located LLM claims collapse to one group and only genuinely independent
   origins compose by noisy-OR. The dead-code gap is closed: the defense ADR-11
   specified becomes a live caller, proven by a traversal/confidence test where N
   derivatives of one origin do **not** out-corroborate one independent
   observation.

4. **Origin accounting precedes entity-resolution soft-match, and gates
   enrichment.** Entity-resolution soft-merge of near-duplicate entities is
   **blocked on** parts 1–3 (merging before origin accounting manufactures
   inflation). No real enrichment worker ships without the reward-gate control
   plane, which shares this lineage primitive (watermark of what carries which
   origin; per-origin convergence guard) and is **co-designed on the same
   metadata substrate** rather than bolted on.

**Scope of the first cut (the tradeoff).** Parts 1–2 are origin-keyed provenance
+ a per-origin ceiling: structural, cheap, and exact for the *same-origin
derivative* hazard. Full **lineage-aware clustering** of semantically-correlated
distinct origins (the principled, NP-hard general solution; region-based BP) is
**deferred** — it is the expensive step, off the critical path while traversal is
flat (0.8–2.6 ms to depth 4 on the enriched slice) and only earns its cost once
multi-origin corroboration exists at scale, consistent with ADR-3's deferral of
independence-grouping. The semantic-judgment cost the council flagged lands here,
in the deferred part, not in the first cut.

This **decides** the independence open problem left open in ADR-9 §Consequences
(strength dimension) and operationalizes the deferred read-path wiring of ADR-3
(confidence dimension). It does not supersede either; it resolves their shared
open Consequence.

## Consequences

- **Easier.** Corroboration becomes honest under enrichment: a hallucination or a
  derivative cannot inflate confidence by repetition, in either dimension. The
  ADR-11 typed-grouping defense stops being dead code. A real reward-gated
  enrichment worker can be built on a sound base instead of widening the hazard.
- **Harder.** Two keys now travel with every reinforcement (origin vs emission
  instance); connectors must supply a content-derived origin key — a new
  obligation at the ingest boundary. The read path gains a grouping step it does
  not have today.
- **Deferred, not closed.** Semantically-correlated *distinct* origins still
  over-corroborate until lineage-aware clustering ships; the first cut only
  corrects same-origin derivatives. This is acceptable while the graph is small
  and single-source (0/148 measured) and is explicitly bounded by the entity-
  resolution coupling (part 4).
- **Sequencing is load-bearing.** Entity-resolution soft-match must wait for
  parts 1–3; enrichment must wait for the reward-gate. Violating either order
  manufactures the inflation this ADR prevents.

## Alternatives

- **Leave `combine_typed` deferred; keep the naive chain product.** Rejected —
  the spike showed it is dead code on exactly the path a real enrichment worker
  will make hot; deferring past that point ships the N3 cascade.
- **Keep provenance = emission instance, add no origin key.** Rejected — counts
  derivatives as independent witnesses and admits the immortal-edge hazard; a
  per-source ceiling is meaningless without an origin to key it on.
- **Soft-merge near-duplicate entities first, then account.** Rejected — merging
  the 24 near-dup pairs before origin accounting manufactures correlated-evidence
  inflation (the council's #1 risk). Entity-resolution comes after.
- **Full lineage-aware clustering / region-based BP now.** Rejected as the *first*
  cut — NP-hard, off the critical path while traversal is flat and corroboration
  is single-source; retained as the deferred general solution.
- **A budget breaker as the enrichment control.** Rejected as sufficient — the
  spike saw 564 triggers from one seed contained only by the fuse; a fuse caps
  cost, it does not schedule. The reward-gate is the scheduler and shares this
  substrate (part 4).

## Related

- [ADR-3: Confidence Calculus](0003-confidence-calculus.md) — the confidence-side
  independence rule this wires into the read path.
- [ADR-9: Stigmergic-Loop Stability](0009-stigmergic-loop-stability.md) — the
  strength-side open problem this decides.
- [`confidence-calculus.md` → strength dimension](../architecture/confidence-calculus.md#independence-in-the-strength-dimension)
  — the solution space (provenance contract / per-source cap / lineage-aware) this
  chooses among.
- swarm ADR-11 (zones + claim typing) — defines `combine_typed`; swarm ADR-13
  (entity resolution) and ADR-14 (data/memory model) — the coupled kernel pieces.
- `board/research/cognitive-activation-spike.md` — the code-grounded stress brief.
- `swarm/docs/design/evidence-origin-substrate.md` — the implementable design.
