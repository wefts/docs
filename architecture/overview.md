# Architecture Overview

Swarm is a local-first automation and cognition system built as a
microkernel/ports-and-adapters architecture. The public kernel owns the stable
control plane; private hives own concrete deployment choices, plugins,
integrations, secrets, and data.

The point of the split is simple: the core can be public and reviewable without
dragging real integrations or private state into the same repository.

## Workspace Shape

The top-level directory is a workspace, not a product repo:

```text
swarm/
  docs/       shared architecture, standards, vocabulary
  swarm/      public kernel/control-plane repo
  hive/       private instance/deployment repo
  scripts/    local operator scripts, outside git
  .mcp.json   local agent/tool wiring
```

Each project owns its own scratch directory:

```text
swarm/tmp/
hive/tmp/
```

There is no shared top-level `tmp/`. Scratch belongs to the project whose work
created it.

## Repository Roles

`docs/` is the cross-repo canon. It defines architecture rules, vocabulary,
guardrails, and verification policy. It should stay independent of any single
machine or private integration.

`swarm/` is the public kernel repo. It owns the stable runtime, protocols, port
contracts, storage substrate, and implementation docs. It must not contain real
plugins, private corpora, secrets, or environment-specific orchestration.

`hive/` is a private instance repo. It owns `docker-compose.yml`, `.env.example`,
plugin source while plugins are still instance-local, data roots, and pointers to
secrets. A hive may become public later, but it should be designed as private by
default.

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

## Shipping Model

The default development model is a small polyrepo workspace:

- public `swarm/` for the kernel;
- private `hive/` for a concrete instance;
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
- Plugin manifests use the `<domain>_<kind>` naming rule.
- Remote synchronization is an operator action, not an agent default.

## Related Docs

- [ports.md](ports.md) defines the extension-point model.
- [../standards/guardrails.md](../standards/guardrails.md) defines safe operating
  boundaries.
- [../standards/verification.md](../standards/verification.md) defines how changes
  are checked.
- `swarm/docs/system_architecture.md` contains kernel-specific implementation
  detail.
- `hive/README.md` contains private instance conventions.
