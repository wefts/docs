# Current State

The shortest honest answer to "what is true right now?" — read this first, update
it last. Not a roadmap, not a task list. When this file and an older doc disagree
about what is current, this file wins; when this file and the code disagree, the
code wins and this file is stale.

## Identity (settled)

Three names, three jobs. Keep them distinct in all docs and code:

- **`wefts`** — the GitHub namespace and workspace root: the unit that ties
  `docs/`, `swarm/`, and `hive/` together. It is *not* a product, a single repo,
  or a deployable thing.
- **Swarm** — the product: the local-first heterogeneous cognitive system. Lives in
  the public `swarm/` kernel repo.
- **Hive** — the environment: a concrete deployment instance (compose, plugins, env,
  data roots, secrets pointers). Lives in the private `hive/` repo.

This naming is settled. The shared `docs/` tree uses this distinction throughout;
repo-specific docs may still need occasional spot checks.

## Repositories (current)

The top level is a `wefts` workspace, not a repo. Inside it:

```text
wefts/                 (workspace; org = wefts; top level is NOT a git repo)
  docs/    git, PUBLIC   shared canon — this repo
  swarm/   git, public   the product: kernel/control-plane (intended public)
  hive/    git, PRIVATE  the environment: instance/deployment
  scripts/ no git        local operator tooling (sync, env), outside git by design
```

The three repos are split apart and `docs/` is public; the workspace shape is settled.

## What is canonical today

`docs/` holds the shared canon and is in good shape:

- `standards/` — guardrails, verification, workflow, conventions, code-style,
  how-to-write-adr. All present and current.
- `architecture/` — overview, ports, confidence-calculus.
- `decisions/` — **workspace** ADR-0..12, indexed, anchored, all records `Complete`.
  The **swarm-local** sequence ADR-1..14 lives in `swarm/docs/decisions/` (a separate
  numbering; ADR-13 entity resolution, ADR-14 data/memory model are the newest).
- `reference/` — glossary and bibliography (both substantial, annotated, sourced)
  and concepts. The glossary now carries the shipped memory vocabulary (§8: content /
  chunk / hybrid retrieval / RRF / relevance floor / answerability).
- `guide/` — plain-language guides (memory model, search, the graph).
- `README.md` (authority map) and this `STATE.md`.

## In flight / known gaps

- **Naming propagation is mostly done.** The shared `docs/` tree uses
  `wefts` / Swarm / Hive consistently. Remaining spot checks belong to
  repo-specific docs in `swarm/` and `hive/`.
- **Guardrail enforcement is partial.** Hooks cover only the 🔒 Never tier. The
  Ask-first tier still rests on discipline — the guard script that would harden it
  is not written.
- **Instruction-file migration is mostly done.** Root `AGENTS.md`,
  `swarm/AGENTS.md`, and `hive/AGENTS.md` are canonical. `CLAUDE.md` files
  should remain pointers, not second sources of truth.
- **The cognitive-swarm thesis is now BUILT (reward-gated), not just demonstrated.**
  The cognitive-activation spike (`board/done/cognitive-activation-spike`) proved the machinery works
  when forced (7.9 triples/node; converging loop; `node.vec`'s first consumer). The evidence
  accounting that was dead code then (`combine_typed` unwired) is **wired** (ADR-13 substrate), and a
  real **reward-gated enrichment worker** is now **built and validated** (`board/done/enrichment-worker`):
  it extracts S-P-O claims and writes them as `claim`-kind assertions onto the sound evidence base
  (they corroborate honestly, derivatives don't inflate). Validated end-to-end on the slice
  (qwen3:14b, 8 claims/node/110 s, then wiped). It is **off by default** — enrichment is the
  cost-asymmetry pillar (~120 s/source), fired deliberately by a bounded worth-it scan, never
  continuously. So the cognitive differentiator is **reachable and built**, awaiting a deliberate
  corpus run + threshold calibration to become load-bearing; today's honest identity remains
  excellent private hybrid-retrieval-over-graph with answerability, now with cognition on tap.
- **ADR-13 evidential origin — SHIPPED end-to-end (the standing correctness debt, now paid).**
  ("ADR-9 evidential-origin" in older notes = the open problem; decided as **workspace ADR-13**,
  Proposed.) The evidence/metadata substrate is built and verified in swarm
  (`board/done/evidence-origin-substrate`):
  **origin** is a first-class reinforcement property; `seen_count = count(distinct origin)` (a
  same-origin derivative neither reinforces nor refreshes the decay clock — immortal-edge hazard
  closed); connectors thread a stable origin at the ingest boundary (silent fallback made
  observable); `combine_typed` is **wired** via node-local `Swarm.Graph.Corroboration` (was
  zero-caller dead code) — independent origins corroborate (noisy-OR), co-located claims collapse,
  structural edges excluded; the per-origin reinforcement ceiling **is** distinct-origin counting.
  The reward-gate control plane is **designed** (`swarm/docs/design/enrichment-control-plane.md`)
  with its **build deferred** to the `enrichment-worker` epic (no worker exists yet — building the
  gate's tables now would be dead code). Method: each step got code review + a 2-family decorrelated
  council (codex + local gemma) before proceeding. **Unblocks** `entity-resolution` soft-match (Y).
- **Answer-path quality gaps (carded, localized).** `key-arm-answerability` (stub-title
  out-of-scope leak), `first-person-false-ownership` ("my" in a title); the absolute
  relevance floor's relative-gate calibration + paraphrase-MRR tuning live in
  `node-vec-per-type`. None structural.

## Architecture direction (stable)

A local-first heterogeneous cognitive swarm: cheap specialized processes run
continuously; expensive models are rare, deliberate escalations; coordination
happens through a shared graph, not direct agent-to-agent messages; the public
kernel stays small; concrete integrations live outside it behind typed ports. Full
detail in `architecture/overview.md` — not repeated here.

## Boundaries (stable)

- Secrets never live in `docs/` or `swarm/`, nor in committed `hive/` files.
- Private data lives only in `hive/`.
- The kernel never imports plugin source as a hidden dependency.
- Remote sync is a human/operator action, never an agent default.
- Each repo owns its own `tmp/`. Planning lives in the workspace-root `board/`
  (Kanban: `ideas/ todo/ doing/ done/` + `research/` + `journal.md`) — its own
  **local git repo with no remote** (a backup, never pushed → never leaks).

## Recently shipped

- **Entity-resolution soft-match — built + validated (Y, unblocked by ADR-13)**
  (`board/done/entity-resolution`; swarm ADR-13 §3.2). The semantic-fragmentation gap's last layer:
  **ER-1** entity identity vectors (embed the key — worker-minted entities have no body); **ER-2**
  candidate proposal under a TWO-signal hard gate (vector cosine AND lexical token-Jaccard — cosine
  alone over-proposes and a false merge contaminates evidence, which origin accounting can't undo);
  **ER-3** a conservative LLM confirm (precision-first; any doubt ⇒ no merge) → origin-safe
  `merge_nodes`, with an audit trail of every decision (ids/scores, no content). **Validated on the
  slice** (bge-m3 + qwen3:14b): a true dup merged (cosine 0.962), a distinct pair did not — precision
  OK; wiped. Off by default.
- **Enrichment worker — the reward-gated cognitive layer, built + validated**
  (`board/done/enrichment-worker`; ADR-13 / EOS-4). Ran as a `wefts-campaign`, one card at a time
  with code review + a 2-family council each: **EW-1** edge-level `evidence_kind` (the assertion
  carries its kind, not the source node — refines EOS-2); **EW-2** the worker (extract S-P-O → claim
  assertions, origin = source, prior reliability 0.5); **EW-3** content-sensitive watermark +
  stale-claim replacement (a write failure aborts the run); **EW-4** worth-it priority scheduler
  (novelty gate + centrality + criticality; `explain/2` audit); **EW-5** bounded worth-it scan +
  generation-bounded convergence + per-candidate row lease. **Validated end-to-end on the slice**
  (qwen3:14b, 8 claims/node/110 s, then wiped to baseline). Off by default (cost-asymmetry). The real
  run caught a bug mocks couldn't (a connection held across the 120 s LLM) — fixed with a row lease.
- **ADR-13 evidential origin — the evidence/metadata substrate, shipped end-to-end**
  (`board/done/evidence-origin-substrate`; workspace ADR-13 Proposed; spec
  `swarm/docs/design/evidence-origin-substrate.md`). Ran as a `wefts-campaign`, one card at a time,
  each with code review + a 2-family decorrelated council (codex + local gemma) BEFORE proceeding:
  **EOS-1** origin first-class + `seen_count = count(distinct origin)` (schema v3→v4); **EOS-1b**
  connectors thread a stable origin at the ingest boundary, fallback made observable; **EOS-2**
  `combine_typed` wired via node-local `Swarm.Graph.Corroboration` (was zero-caller dead code;
  structural edges excluded, origin-dedup before combine); **EOS-3** the per-origin reinforcement
  ceiling IS distinct-origin counting (verified, canon reframed); **EOS-4** reward-gate control-plane
  **design** (build deferred — no enrichment worker exists yet, so the gate's code lands with the
  `enrichment-worker` epic, not before). Defenses actually wired + tested (N derivatives of one
  source no longer over-corroborate, in both the strength and confidence dimensions). Full suite
  green throughout (212/0 at peak), format + credo clean. All repos `main`, **not pushed**.
- **Cognitive-activation spike — the cognitive thesis tested on the live slice (guarded,
  disposable), and ADR-9 profiled from measurement.** A research/spike campaign
  (`board/done/cognitive-activation-spike`, note in `board/research/`) forced the dormant
  cognitive layer to fire under a hard guard (snapshot → activate → measure → **wipe to the
  verified snapshot**). Found: reward-gated **enrichment was a concept with no implementation**
  (the LLM engine exists; built a disposable extractor → 7.9 useful S-P-O triples/node, 0 parse
  failures); on 30 intranet sources it wrote 150 claims / 239 entities / 148 single-source-capped
  claim edges. A 2nd stigmergy reactor coexisted; the worker→graph→worker loop **converges**
  (proven deterministically — enrichment output {entity} ⊥ input {article}); `node.vec` got its
  **first consumer** (soft entity-resolution surfaced **24 near-dup pairs** the exact key missed).
  ADR-9 stress: exact-(s,p,o) corroboration **0/148** — over-corroboration is **semantic and
  coupled to entity-resolution**; plus two code-level certainties (`combine_typed` unwired,
  `seen_count` ⊥ `reliability`). Traversal stayed flat **0.8–2.6 ms** to depth 4. Council (gemini-
  3.1-pro + local gemma4, BOTH SOUND-WITH-CAVEATS) added a coupled track: a reward-gating +
  watermark + priority **control plane** (a budget fuse is not a scheduler). Disposable tooling in `hive/`
  (`main`, **not pushed**); **no enriched state persists**. **Unblocks ADR-9 as the next epic.**
- **data-impl Phase 2 — structure-aware segmentation + weighted-RRF ranking; the
  data-foundation epic is COMPLETE.** Card 6: a canonical Markdown body profile
  (`swarm_markdown_v1`) + a structure-aware kernel segmenter (headings → sections; code
  blocks + pipe tables ATOMIC, never flattened) replaces the prose-only one; connectors
  emit it (Confluence XHTML→md preserves tables/code that were previously DISCARDED;
  MediaWiki headings→ATX). Verified on the slice: structure survives into chunks (430
  table-/719 code-/3907 heading-bearing group chunks), no Wikipedia regression, and an
  ablation proved the segmenter loses no information. Card 7: measured the arms separately
  — the **dense arm is essential** (paraphrase NL queries: lexical ~0–3% → hybrid ~72%
  recall@5 on *both* source shapes), but finer chunks let a dense "magnet" demote exact
  lexical hits (ablation: 100% of misses were demotions, 0 missing). Fix = **weighted RRF**
  (`config :swarm, :retrieval, lex_weight: 3.0`): floors exact keyword hits without
  touching paraphrase ranking (paraphrase has no lexical rows). Result: group verbatim
  hybrid recall@5 94.7→100% / MRR 0.687→0.878, public MRR 0.966→0.99, paraphrase held.
  Decorrelated council (gemma + qwen3 convergent; codex host-flaky). `node.vec`
  aggregate-vs-identity DEFERRED — it is write-only (no consumer reads it → unmeasurable;
  `board/todo/node-vec-per-type`). swarm `main` carries it (segmenter + weighted-RRF feat
  branches ff-merged), **not pushed**. Spec §2/§5 `sync-specs`'d to match.
- **Campaign A — real Confluence + intranet MediaWiki connectors, live-verified; the
  Phase-1 metrics de-risked off clean prose.** Two self-contained `hive/plugins`
  connectors (private repo) implement the ADR-5 `fetch/2` contract and are auto-loaded by
  `Swarm.Plugins` as `"confluence"` + `"mediawiki"` — the in-process dev-adapter mode
  (ports.md, ADR-11); no kernel change, source-specific parsing kept out of the public
  kernel by design. Confluence: HTTP Basic, CQL search endpoint, **opaque `_links.next`
  cursor** (a real-data bug — the endpoint silently ignores a manual `start` offset and
  re-returns page 1; the de-risk *found and fixed* it), storage-XHTML→prose, `<ri:page>`
  links + ancestors `child_of`, label/length skip, `group` scope. MediaWiki: allpages +
  `continue` pagination, wikitext→prose, link + redirect resolution, best-effort
  BotPassword login (degrades to anon), `group` scope. Both kernel-driven for completeness
  (`truncated?` on a ceiling, never silent). Hermetic tests 7/7 + 8/8, credo/format clean,
  live-smoke + a multi-source live slice (`hive/scripts/`). **Re-measured on a 3-source,
  mixed-scope `swarm_slice`** (2345 public Wikipedia + 821 group intranet + 600 embedded):
  recall@5 **HOLDS on the messy corpus — 98.7% (intranet) vs 100% (Wikipedia), answerability
  100%**; content-over-title lift is *larger* on real org data (+84pp vs +64pp). Scope
  isolation held (intranet never `public`). Traversal cost flat ~2 ms to depth 4 (ADR-3
  wall not triggered; hubs shallow so still under-tested). **New honest gaps, carded:**
  fragmentation 0→3 same-scope groups (the predicted entity-resolution soft-match case,
  `todo/entity-resolution`); hybrid **MRR degrades on messy data** (0.77 vs lexical 0.989) —
  the dense arm reorders exact hits down, a Phase-2 fusion-tuning signal needing a
  paraphrase probe set (`todo/hybrid-rank-on-messy`). Verdicts from a decorrelated council
  (codex gpt-5.5 + local gemma, independently convergent). hive `main` carries it (feat +
  fix + chore branches ff-merged), **not pushed**. This **unblocks data-impl Phase 2**
  (`data-impl-segmenter`, `data-impl-vector-recall`) — the ≥2-source-shape gate is met.
- **Data-foundation epic Phase 1 — ADR-14 / C1′ is BUILT, live-verified, and the answer
  path now retrieves content.** Five cards (swarm): (1) a stateless `content`/`chunk`
  side-store (FK CASCADE, HNSW + GIN-FTS, scope read via `node.scope`); (2) a closed
  kernel **node-type vocabulary** (`Contract.types/0`, fail-loud; edge/relation types stay
  open), schema stamped v3; (3) no-LLM ingest that populates `content` + `chunk` +
  aggregated `node.vec` (prose strip → segmenter → `Content.put_body` in-tx → `Embedder`
  worker off-tx) — closes the structure-not-prose gap; (4) scope-aware per-kind
  `merge_nodes` (cross-scope **refused**, chunk-span union + content survivorship +
  `node.vec` re-aggregate) + a reversible standing **`node_alias`** table consulted by
  `upsert_node`; (5) **hybrid retrieval** `Retrieval.search` — lexical(tsvector/GIN) ∥
  dense(pgvector HNSW) RRF-fused, scope on both arms, → recursive-CTE traversal, cited
  spans always. Then a **relevance floor**: the dense arm's absolute cosine gates each
  chunk (lexical hit OR cosine ≥ floor) before RRF, so the system can return `:not_found`
  for out-of-scope queries; `Core.ask` now routes through this hybrid path (owner "my X"
  stays key-based, T8 preserved). Verified on a live `swarm_slice` (real bge-m3, ~96
  Wikipedia pages): answerability 0%→~100% (floor), recall@1 ~30%→98%, recall@5 100%,
  fragmentation 0, retrieval p50 ~190ms (≈ the query-embed round-trip). Checked by a
  decorrelated critic council + **multi-agent manual QA** (privacy/scope no-leak, content
  recall, cross-lingual UA/FR, paraphrase all pass). A KEY diagnosis: the early "embedding
  hubness" / "no recall lift" symptoms were a measurement+fusion artefact (RRF discarded
  the absolute cosine), not the embeddings. Honest open gaps, all carded in `board/todo/`:
  key/title-arm out-of-scope leak (~2/10 stub-title matches), `:swarm,:retrieval` floor is
  absolute (relative gate is the Phase-2 calibration), `"my"`-in-title false-ownership,
  and the embed worker is wired but the deployed hive image predates it. **Phase 2**
  (source-adapted segmenter, per-type aggregate-vs-identity vec) is now **UNGATED** —
  Campaign A delivered the ≥2-source-shape requirement (above). swarm `main` carries it
  (3 feat branches ff-merged), **not pushed**.
- **Data-foundation research epic — the memory model is decided (no code).** A
  research-first epic (`board/research/data-foundation-research.md`) ran in 4 steps:
  landscape survey (3 decorrelated web-grounded agents → `board/research/data-landscape.md`),
  candidate models (`data-options.md`, 4 distinct), a **3-family decorrelated consilium**
  (codex / gemini-3.1-pro / local glm-4.7-flash — SOUND-WITH-CAVEATS ×2 + a FLAWED whose
  fix *was* the converged fix), and the write-up. Outcome **swarm ADR-14 (Proposed)** +
  `swarm/docs/design/data-memory-model.md`: **C1′** — a coarse, lineage-bearing graph node
  (1 source ≈ 1 origin; sole evidence-bearing tier) over a **stateless content/chunk
  side-store** (chunks = HNSW-indexed retrieval handles, never evidence-bearing); node
  vector is an aggregate/identity vector (not a full-doc mean; source-adapted segmentation,
  bge-m3 8192 limit); retrieval = hybrid lexical+dense (RRF in SQL) → native graph
  traversal; normalization = the ADR-13 funnel + two new seams (kernel-owned **type
  vocabulary**, **scope-aware merges**); claim/triple extraction is **reward-gated
  enrichment**, never the continuous default. Lint-clean; consilium recorded in
  `board/journal.md`. **Unblocks** `board/todo/ingest-persist-content` as a build campaign.
  No implementation code written. (A `scripts/gemini_review.sh` wrapper was added so the
  Gemini key is sourced at script runtime, never in agent context.)
- **Phase E — first live slice (the kernel parses real data at last).** One campaign
  (`board/done/phaseE-live-slice`): E1 reconsolidated `swarm/docs/system_architecture.md`
  into a top-down story with all 8 Phase B/C/D subsystems, an admitted control plane,
  and an honest "contract-vs-proven-on-real-data" §12 (codex-reviewed). E2 built a
  public **Wikipedia/MediaWiki connector** (`fetch/2`, allpages + `continue`, wikitext
  strip, link graph) — 9 tests — and ran a real ingest→graph→ask loop. **The entire
  T0–T13 marathon was deployed to the conditional hive prod for the first time** (the
  running image was pre-marathon MVP); a live run put 1313 article nodes / 1379 edges
  into prod `swarm_dev` and answered via the deployed Core API (self-model + graph
  retrieval). The predicted failure modes are now **confirmed on real data**: entity
  fragmentation (**now fixed**, swarm ADR-13 Accepted: layer 1 URL-decode +
  layer 2 kernel `merge_nodes/3` provenance-preserving merge + MediaWiki redirect
  resolution at ingest — the live fragmentation probe dropped 4→0 on real data),
  off-topic deflection collapses to escalate when ML is down, consilium escalation
  latency >280s (motivates swarm ADR-1), and ingest persists structure not prose
  (no node `vec`/content). Full suite 148/0; format/credo/markdownlint clean; codex
  consilium on both load-bearing docs. swarm ADR-13 (entity resolution, Proposed)
  added. Changes in the `swarm/` working tree pending an operator commit; not pushed.
- **Phase D (T10–T13) — mechanisms & hardening.** T10 poison/DLQ + demand-pull
  backpressure (swarm ADR-9). T11 decay-driven trace GC + bounded weights +
  re-derivable ρ (swarm ADR-10, OP#1). T12 graph zones + claim/observation typing
  with reward-gated persistence — the N3 defense (swarm ADR-11; schema v1→v2). T13
  stagnation monitor + deduped bystander surface + wired watchdog (swarm ADR-12).
  All four critic-verified (three hit FLAWED first, re-critiqued after fixes,
  resolved); 139 tests 0 failures, gates clean. Follow-ups carded
  (`reward-quorum-and-audit`, `stagnation-escalation-sink`).
- **Phase C (T5–T9) — answers, cost, channels, identity.** T5 LLM per-escalation
  budget (ceiling + model-boundary backstop + cost telemetry; the 385k path is
  refused) → ADR-7 budget. T6 answer-result algebra (found/not_found/partial/error,
  typed on the wire) → swarm ADR-6. T7 presentation-determinism standard (kernel
  emits facts, channels render; CLI renders status + verbatim values). T8 kernel
  self-model (real inventory/freshness/capabilities) + asker identity (`viewer`
  resolves "my X", scoped) → swarm ADR-7. T9 off-topic deflection (tier0, zero-LLM
  for recognized off-topic) + `skill` port → swarm ADR-8; chat channel + persona
  copy deferred to `board/todo/hive-chat-channel`. All five critic-verified; 118
  tests 0 failures, credo/dialyzer/format clean, CLI 7/0.
- **Phase B (T3+T4) — connector ingestion contract.** swarm ADR-5: the
  `connector` port gains a kernel-driven `fetch/2` paginated pull, and
  `Swarm.Connector.Sync` owns completeness (drives the cursor to exhaustion,
  retries flaky fetches, logs source ceilings instead of silently capping, tracks
  the delta watermark, reconciles against a declared total, and returns a report —
  never raw payloads, so a model only ever reads graph state). Proven by a
  hostile-fixture connector + an 8-test completeness gate. `mix test` 95/0,
  credo/dialyzer/format clean; critic SOUND-WITH-CAVEATS (fixes applied). The real
  Confluence/Mediawiki glpi port is the follow-up
  `board/todo/confluence-mediawiki-connectors` (hive). `board/done/T3…`,`T4…`.
- **T2 — graph-integrity contract.** The shared graph schema is now
  write-validated at the `Swarm.Graph.Store` boundary (swarm ADR-4, Accepted):
  type/scope vocabulary, the ADR-5 visibility invariant (edge scope ≤ narrowest
  endpoint — moved from ingest into the kernel, closing the public-edge-between-
  private-nodes leak), reliability range, provenance shape. Defense-in-depth: DB
  CHECK on scope vocab + `FOR SHARE` on the endpoint read; schema version stamped.
  `mix test` 87/0, credo/dialyzer/format clean; critic-reviewed SOUND-WITH-CAVEATS
  (caveats applied). Provenance *lineage* stays ADR-9's open problem; durability
  follow-up `board/todo/graph-rescope-and-trigger`. `board/done/T2…`.
- **T1 — confidence-saturation spike.** Measured where confidence collapses on a
  dense graph: it is **path enumeration** (the recursive-CTE traversal), not the
  independence-grouping open problem (grouping is ~O(P), sub-second even at 299k
  groups). Decision (swarm ADR-3, Proposed): node-bounded best-conf-per-node
  relaxation + best-effort-above-budget; region-based BP deferred (off critical
  path). Bench `swarm/kernel/bench/confidence_saturation.exs`; critic-verified
  (SOUND-WITH-CAVEATS, equivalence independently confirmed). Impl follow-up:
  `board/todo/traverse-relaxation`. `board/done/T1…`.
- **T0 — stub ADRs transplanted.** All eight `Stub` records in `docs/decisions/`
  (ADR-1,2,4,5,6,7,8,9) now carry the full accepted wording from
  `swarm/docs/swarm_architecture_spec.md`, grounded in the built kernel and
  verified by independent critics (faithful transplant, no invention, open
  problems OP#2/#4 + ADR-9 correlated-events kept open). `board/done/T0…`.
- **MVP vertical slice** built and canonical in code (`kernel/`, `ml/`, `cli/`):
  graph substrate, gate, consilium, ingest, embeddings, CLI (old `swarm/tmp/tasks`
  01–08; see `board/done/mvp-build`).
- **Dockerization** — full containerized stack (DHI images, `hive/` compose with
  **GPU Ollama**, proven offline boot, digest-pinned 3-tier registry, HA replicas);
  `swarm/docs/design/dockerization-design.md`, `hive/docs/operations.md`.
- **Stigmergy signal** — graph-write → worker reaction (outbox → tailer → dispatch →
  per-key lanes). swarm-local **ADR-2** (Accepted) + spec; `board/done/stigmergy-signal`.
- **`swarm/infra/` → `swarm/dev/`** rename; **model-residency scheduler** concept
  (swarm-local **ADR-1**, Proposed). `swarm/` now carries repo-local ADRs in
  `swarm/docs/decisions/`, distinct from the workspace `docs/decisions/`.

## Next

The full roadmap is `board/roadmap.md`; task cards in `board/todo/`; rationale in
`board/research/`. The T0–T13 sequence, Phase E, the data-foundation research epic,
**data-impl Phase 1 + 2**, **Campaign A (real connectors)**, the **guarded cognitive-activation
spike**, the **evidence-origin substrate (ADR-13, X)**, the **reward-gated enrichment worker**, and
the **entity-resolution soft-match (Y)** are all shipped + verified. The evidence accounting that was
dead code is wired; claims corroborate honestly; duplicate entities fold without inflating. **The
foundation AND the cognitive layer are feature-complete — off by default.**

The forward cut (after enrichment + entity-resolution landed, 2026-06-25 — journal):

1. **Turn it on, deliberately, then calibrate.** A real corpus run on the slice (bounded worth-it
   enrichment scan + an entity-resolution pass), then **calibrate** the priority weights/threshold and
   the ER vector/lexical thresholds (ADR-8) from the logged decisions (`Priority.explain`,
   `entity_resolution_audit`) — defaults are heuristic, not yet calibrated. Re-measure the
   fragmentation probe → 0 on the real intranet corpus. Promote **ADR-13 → Accepted** once confirmed.
2. **`traverse-relaxation`** stays deferred (traversal flat 0.8–2.6 ms to depth 4); trigger is real
   enrichment **at scale** — now reachable once enrichment is turned on. `node-vec-per-type` unparked.
3. **Opportunistic:** `key-arm-answerability`, `first-person-false-ownership` (localized answer-path).
