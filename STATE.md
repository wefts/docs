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

The three repos were **just split apart**, and `docs/` was **just made public**
(it defaulted to private on creation). Bringing all three into consistent order is
the work in progress right now.

## What is canonical today

`docs/` holds the shared canon and is in good shape:

- `standards/` — guardrails, verification, workflow, conventions, code-style,
  how-to-write-adr. All present and current.
- `architecture/` — overview, ports, **confidence-calculus (just added)**.
- `decisions/` — ADR-0..12. Present, indexed, anchored, and **all records now
  `Complete`**: the accepted wording is transplanted from the old spec (T0).
- `reference/` — glossary and bibliography (both substantial, annotated, sourced)
  and concepts.
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

The full roadmap is `board/roadmap.md` (phases + status map + the glpi-agent
connector fold-in); task cards in `board/todo/`; rationale in `board/research/`.
In order:

1. **The roadmap T-sequence (T0–T13) is complete** — phases A (contracts), B
   (connectors), C (answers/cost/channels/identity), D (mechanisms/hardening) all
   shipped. What remains is the **follow-up backlog** in `board/todo/`: the real
   glpi connectors (`confluence-mediawiki-connectors`), kernel hardening
   (`traverse-relaxation`, `graph-rescope-and-trigger`, `reward-quorum-and-audit`,
   `stagnation-escalation-sink`, `upsert-node-emit`), the hive chat channel, and
   cross-cutting (naming spot-check, Ask-first guard script). Re-cut priority with
   the `architect` skill before the next campaign.
2. The real connector implementation: `todo/confluence-mediawiki-connectors`
   (port Confluence + Mediawiki from `~/Code/glpi-agent` behind `fetch/2`, hive).
3. Kernel follow-ups teed up: `traverse-relaxation` (impl swarm ADR-3),
   `graph-rescope-and-trigger` (durability for the ADR-4 visibility invariant).
   The ADR-9 strength-independence decision (provenance evidential-origin) is still
   the biggest open correctness question.
4. Cross-cutting: naming spot-check in `swarm/`/`hive/`; the Ask-first guard script
   (promote 📝 → 🔒).
