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
- Each repo owns its own `tmp/`; there is no shared top-level `tmp/`.

## Next

In order:

1. Transplant the full accepted ADR wording into `Stub` records in
   `docs/decisions/` (mostly ADR-1..9).
2. Spot-check repo-specific docs for stale `wefts` / Swarm / Hive naming.
3. Write the Ask-first guard script and promote the high-stakes 📝 lines to 🔒.
