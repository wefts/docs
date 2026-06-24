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

This naming was just decided. The shared `docs/` tree now uses this distinction;
repo-specific docs may still need spot checks.

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
  and concepts. (The shipped memory vocabulary — *content / chunk / hybrid retrieval /
  answerability* — is not yet in the glossary; minor follow-up.)
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
- **Answer-path quality gaps (carded, from the data-impl knowledge-test).** The
  retrieval/answer layer is validated only on ONE clean source so far. Open in
  `board/todo/`: key/title arm can false-`found` a content-less stub by exact title
  (`key-arm-answerability`); `"my"` inside a title misroutes to the identity path
  (`first-person-false-ownership`); the relevance floor is absolute (a relative gate
  is the Phase-2 calibration); the dense arm lowers MRR on exact-keyword probes (RRF
  fusion weighting → `data-impl-vector-recall`). All localized, none structural.
- **Data-impl Phase 2 is gated on a second, messy real source.** The source-adapted
  segmenter and per-type aggregate-vs-identity vector measurement need ≥2 source shapes;
  they wait on `confluence-mediawiki-connectors` (access now unblocked).

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
**data-impl Phase 1**, and now **Campaign A (real connectors)** are all shipped.

1. **A — DONE.** Real Confluence + intranet MediaWiki connectors shipped and
   live-verified (see Recently shipped). Recall held on the messy corpus; the de-risk
   surfaced two carded gaps (fragmentation soft-match; hybrid MRR on messy data) and
   measured traversal cost flat (B not promoted).
2. **data-impl Phase 2** (now UNGATED): `todo/data-impl-segmenter` (source-adapted
   segmentation, now ≥2 source shapes to tune against) + `todo/data-impl-vector-recall`
   (per-type vec, RRF/relative-floor tuning) — the latter now informed by
   `todo/hybrid-rank-on-messy` (dense arm reorders exact hits down on real content).
3. **B — `todo/traverse-relaxation`** (impl swarm ADR-3): stays deferred — A's
   re-measurement showed traversal cost flat (~2 ms to depth 4), not climbing; promote
   only if a deeper/denser region exercises the path-enumeration wall.
4. **D — ADR-9 evidential-origin** stays the biggest open correctness question; promote
   when multi-source ingest starts compounding corroboration (now live — watch it).
   Localized answer-path fixes (`key-arm-answerability`, `first-person-false-ownership`)
   slot in opportunistically.
