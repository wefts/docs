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
- `decisions/` — ADR-0..12. Present, indexed, and anchored. Some records are
  marked `Stub`: the decision is accepted, but the markdown record still needs
  full wording transplanted from the old spec.
- `reference/` — glossary and bibliography (both substantial, annotated, sourced)
  and concepts.
- `README.md` (authority map) and this `STATE.md`.

## In flight / known gaps

- **ADR text is not fully migrated.** `docs/decisions/` exists, but several ADR
  records are still marked `Stub`. The remaining work is to transplant the full
  accepted wording from the old `swarm_architecture_spec.md`.
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

1. **T0 — transplant the stub ADRs** (`docs/decisions/` ADR-1,2,4,5,6,7,8,9 are
   still `Stub`): fold the accepted wording from `swarm/docs/swarm_architecture_spec.md`.
   This is the doc-sync debt, and it gates the contract-first tasks.
2. **T1–T2** — confidence-saturation spike; graph-integrity/provenance/idempotency
   contract → then T3/T4 (connector contract + reference connector).
3. Cross-cutting: naming spot-check in `swarm/`/`hive/`; the Ask-first guard script
   (promote 📝 → 🔒).
