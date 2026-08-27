# Architecture Overview

`wefts` is the workspace that ties the shared docs, the public Swarm kernel, and
the public Hive deployment scaffold together. Swarm itself is a local-first
automation and cognition system built as a microkernel/ports-and-adapters
architecture. The public kernel owns the stable control plane; hive instances own
concrete deployment choices, plugins, and integrations — their secrets and private
data live only outside git (gitignored env/secrets files, volumes).

The point of the split is simple: the core can be public and reviewable without
dragging real integrations or private state into the same repository.

## Workspace Shape

The top-level directory is a `wefts` workspace, not a product repo. The locked
workspace split is [ADR-10](../decisions/0010-wefts-workspace-split.md):

```text
wefts/
  docs/       shared architecture, standards, vocabulary
  swarm/      public kernel/control-plane repo
  hive/       public instance/deployment repo (scaffold, not private data)
  scripts/    local operator scripts, outside git
  .mcp.json   local agent/tool wiring
```

The folder name is machine-local. Never hardcode the absolute workspace path in
product code. Use relative repo paths (`docs/`, `swarm/`, `hive/`, `scripts/`)
from the workspace root.

## Repository Roles

`docs/` is the cross-repo canon. It defines architecture rules, vocabulary,
guardrails, and verification policy. It should stay independent of any single
machine or private integration.

`swarm/` is the public kernel repo. It owns the stable runtime, protocols, port
contracts, storage substrate, and implementation docs. It must not contain real
plugins, private corpora, secrets, or environment-specific orchestration.

`hive/` is a **public** instance repo. It owns `docker-compose.yml`, the layered `env/`
config (ADR-0015), plugin source while plugins are still instance-local, data roots, and
pointers to secrets. Because it is public, no secrets or intranet specifics live in its
committed files — those stay in gitignored `secrets.env`, Docker volumes, and config;
intranet values are parameterized, never hardcoded (`board/todo/hive-publish-readiness-audit.md`).

`scripts/` contains local synchronization and operator tooling. These scripts are
for this workspace and should not be treated as part of the public product.

## Runtime Model

The kernel is the control plane:

- schedules and supervises work;
- owns typed ports;
- reads and writes through stable storage abstractions;
- calls model and plugin adapters through explicit contracts.

Plugins are the data plane:

- connectors ingest external systems;
- tools perform bounded actions;
- workers run background reasoning or maintenance loops;
- channels expose user-facing interaction surfaces;
- model adapters provide inference and embeddings.

The kernel should know *what kind* of capability it is calling. It should not
know the private deployment details of a specific Confluence, Kubernetes cluster,
mailbox, or ticket system.

## Access Model

Access is project-centered, but enforcement stays in the kernel. A Project owns
Sources/Connectors; Project membership derives the actor's effective source scopes. The
graph gate then applies the stable ADR-16 predicate:

```text
visible ⇔ scope ∈ actor.effective_scopes
       AND (owner IS NULL OR owner = actor.id)
```

The fixed admin groups are `Wheel`, `Admins`, and `Staff`. Groups do not grant source
visibility directly; they may be Project members. `superadmin` is a time-boxed elevation
session for local `Wheel` members, not a standing group role. Full detail:
[access-model.md](access-model.md).

## Answer routing: the world-map tier-gate (ADR-17)

`Core.ask` routes through a cost-asymmetry tier chain: cheap **structured serve** from a
continuously-maintained world-map before the expensive consilium. A deterministic
Stage-1 coverage gate (`Swarm.WorldMap.Coverage`) plus a Stage-2 cheap-LLM entailment veto
(`Swarm.WorldMap.Gate`) answer covered asks from graph structure in ~2–4 s instead of the
~55–65 s consilium; anything not clearly covered **escalates** (fail-closed:
`supported=false ⇒ escalate`). A generic `:neighborhood` serve registry
(`Swarm.WorldMap.Domain`) hosts the domains — procedures, `:network` topology, and the
`:who` who-is-who / service / location family — so a new served domain is one registry
entry, not a pipeline edit. The map is built offline (nightly enrichment + connectors) and
served cheap online; served facts stay provenance-carried and re-derivable, never a stale
answer cache. Evidence governance (lineage-aware corroboration, freshness/decay, a serve
contract, identity do-not-merge) sits under it so cheap ≠ dishonest.

## Shipping Model

The default development model is a small `wefts` polyrepo workspace:

- public `swarm/` for the kernel;
- public `hive/` for a concrete instance (scaffold; private state stays out of git);
- optional future plugin repos once a plugin is stable enough to stand alone;
- shared `docs/` for architecture and standards.

Early plugins may live inside `hive/plugins/` because that keeps integration work
close to the environment it depends on. When a plugin becomes reusable, it can be
moved to its own repo without changing the kernel contract.

## Hard Boundaries

- Secrets never live in `swarm/`, `docs/`, or committed hive files.
- Private data never lives in `swarm/`.
- The kernel never imports plugin source as a hidden dependency.
- Third-party tools attach through a port, a manifest, or an adapter process.
- Plugin manifests follow the naming rule in [ports.md](ports.md).
- Remote synchronization is an operator action, not an agent default; the sync
  boundary is [ADR-12](../decisions/0012-operator-sync-boundary.md).

## Related Docs

- [ports.md](ports.md) defines the extension-point model.
- [access-model.md](access-model.md) defines Projects, groups, elevation, and visibility.
- [../standards/guardrails.md](../standards/guardrails.md) defines safe operating
  boundaries.
- [../standards/verification.md](../standards/verification.md) defines how changes
  are checked.
- `swarm/docs/system_architecture.md` contains kernel-specific implementation
  detail.
- `hive/README.md` contains (public) instance conventions.
