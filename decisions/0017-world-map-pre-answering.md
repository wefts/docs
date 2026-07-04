# ADR-17 (workspace): World-map pre-answering — answer from maintained structure, escalate only on insufficiency

## Status

**Proposed (2026-07-04).** Five forks resolved by a decorrelated 2-family blackboard
(codex + gemini); synthesis in `board/research/world-map-blackboard.md`. Design spec →
`swarm/docs/design/world-map-pre-answering.md`; execution epic →
`board/doing/world-map-pre-answering-epic.md`. Item 3 of the post-migration trio;
graduates `board/ideas/world-map-pre-answering.md` (+ its horizon
`board/ideas/world-graph-self-model.md`).

> NB numbering: a **workspace** ADR (spans kernel enrichment/retrieval + a hive connector +
> the shared graph contract), distinct from the swarm-local sequence (whose 0016 is
> pg_search). Builds directly on workspace ADR-13 (evidential origin) and the shipped
> STEP-2 aggregation layer.

## Record Completeness

Complete — direction + all five forks resolved by council; mechanism detail (schemas,
gate signals, RPC shape) goes to the spec.

## Context

- Swarm today = excellent honest retrieval + grounded extraction + a fail-closed judge.
  It answers what a passage **says**, not what a set of passages **implies** about a
  process. Observed live: a routine "how do I do X" — answerable by a human from a bullet
  on one page plus a tool page — makes Swarm retrieve the related pages and then honestly
  fail-closed, because it has no model of the **process**. The knowledge is page-shaped,
  not process-shaped, and the user's vocabulary rarely matches the corpus's.
- A full `ask` escalates to the heavy multi-model consilium (one GPU) → minutes. The fix
  is **not a faster fleet** — it is **not deliberating for the known**.
- The cognitive substrate to answer from structure already exists and is now running:
  reward-gated enrichment (nightly bounded), entity-resolution, STEP-2 knowledge-
  aggregation (answers "what is X" over grouped claim-edges **without reifying facts**),
  evidential origin + content-watermark, the fail-closed `supported` flag. This ADR
  **composes** them into a maintained world-map + a routing gate; it does not rebuild them.
- Prior decision carried in: facts stay canonical claim-**edges**; "deepen understanding"
  is an **aggregation layer, not reification**; statement-node reification stays reserved
  and **must not** be reopened here.

## Decision

Continuously maintain a **world-map** (entities, relations, and **procedures/situations**)
and answer most asks from it via cheap retrieval + aggregation over pre-built structure;
reach the consilium **only** when a routing gate judges the structure insufficient
(genuine novelty / ambiguity / conflict). Think offline (curation), serve cheap online
(cost-asymmetry).

Five forks, decided:

- **A — Representation of procedures.** A procedure is an existing `entity` node plus
  **ordered claim-edges** (`has_step` carrying an integer `step_index`, `requires_tool`,
  `applies_to`), reconstructed by a **read-time aggregation view** that sorts the steps.
  **No node-type vocabulary bump up front** — this honors the no-reification decision and
  keeps the schema tiny; a dedicated `procedure` node kind is a *fallback* only if
  aggregation cannot cleanly separate procedures from plain entities (decide in the spike,
  not up front). **Load-bearing constraint:** the aggregation **must group steps by
  `origin`/provenance before ordering** — else two sources' "step 1" interleave into a
  broken procedure. Steps keep origin + reliability; edges stay reversible.

- **B — The tier-routing gate (the load-bearing fork; both families named it the one that
  sinks the epic if wrong).** A **cheap deterministic structured-coverage gate** decides
  first: are the query's intent slots filled by **citable, current** claim-edges? does a
  procedure candidate exist? is there no unresolved contradiction? are the dependency
  watermarks valid? Only if that passes do we optionally spend a **small-model YES/NO
  sufficiency confirm** (high bar). **Invariant — this protects the trust ADR-16's honest
  judge earned:** `supported = false` ⇒ **escalate**; the structured tier never emits a
  step that is not backed by a current claim-edge. Coverage **count** is explicitly
  rejected as the signal (quantity ≠ sufficiency) — it is slot-filling + citability.

- **C — Bootstrap order.** Fill the map from the **formal corpus first** (the nightly
  enrichment over wiki/runbooks establishes the canonical *intended* process). Ticketing/
  change history (via a `glpi-agent` oracle connector) comes **later**, strictly to surface
  situations / variants / aliases / gaps that map **onto** the formal procedures. This
  **overrides the charter's glpi-first lean** — tickets-first would bake incident noise and
  one-off workarounds into the map as if they were sanctioned process (convergent council).

- **D — Freshness / re-derivability.** Cache **structure, never answers.** A served answer
  is re-aggregated from current edges on every read; a dependency whose content-watermark
  has moved marks the aggregate **stale/unsupported**. There is no answer cache to rot.
  New hazard to close in the spec: **orphaned step-edges on source deletion → "ghost"
  procedures** — the GC/merge path must purge derived step-edges when their source node is
  removed.

- **E — Scope.** A `Self`-as-data node and prediction-as-entities
  (`world-graph-self-model`) are **out of scope** — a separate later proposal sharing this
  substrate. Keeping the epic tight around the gate (B) is what makes it shippable.

## Consequences

- **Positive:** most routine asks answered cheaply from structure; the consilium is
  reserved for genuine novelty; "how do I X" becomes answerable by composing scattered
  facts; vocabulary/synonymy coherence (the carded `concept-synonymy-resolution`, a
  prerequisite substrate step) lets any surface form reach the same concept; the honest-
  judge no-leak/no-fabricate guarantees are preserved because the gate fails closed.
- **Negative / risk:** the gate is the whole bet — too eager to serve ⇒ confident-but-
  incomplete answers that break trust; too eager to escalate ⇒ zero latency/cost win. The
  ADR spends its rigor there. Procedure aggregation can produce broken step-chains if
  provenance grouping is wrong (mitigated by the group-by-origin constraint). Ghost
  procedures on source deletion (mitigated by the GC purge).
- **Reversible?** The representation (aggregation over reversible edges, no vocab bump, no
  reification) is low-commitment and reversible. The gate is additive (a new tier in
  `Ask`, defaulting to today's escalate-everything behaviour until proven). Bootstrap is
  additive (a connector + provenance-marked edges). So the expensive-to-reverse surface is
  small — the durability is in the *invariant* (structured tier fails closed), not in
  frozen data.

## Guardrails carried

No-leak (scope AND owner); ready answers stay provenance-carried + re-derivable (never a
stale cache); one change per repo (connector → `hive/`, enrichment/retrieval/gate →
`swarm/`, this decision → `docs/`); enrichment mutates the graph → snapshot-protected,
gated, measured, like the nightly loop. The cheap-interim consilium-latency work is
orthogonal and may ship anytime.
