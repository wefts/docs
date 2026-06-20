# Concepts

This is the short conceptual model. The longer working vocabulary lives in
[glossary.md](glossary.md).

## Swarm

A swarm is a system made from many small, specialized processes. Each process is
limited, but the system becomes useful because the processes share memory,
observe each other's traces, and escalate only when necessary.

## Wefts

Wefts is the workspace and GitHub org: the weave that ties `docs/`, `swarm/`,
and `hive/` together. It is not a deployable product and not a single repo.

When names are ambiguous, use:

- `Wefts` for the workspace/org;
- `Swarm` for the product/system;
- `swarm/` for the public kernel repo;
- `Hive` for a deployment environment;
- `hive/` for a private deployment instance repo.

## Hive

A hive is a concrete Swarm instance. It contains deployment wiring, env examples,
enabled plugins, data roots, and private integration choices.

The hive is allowed to know about real systems. The public kernel is not.

## Kernel

The kernel is the stable control plane. It owns scheduling, supervision, typed
ports, storage abstractions, guardrails, and the core runtime contracts.

The kernel should be boring, small, and public.

## Plugin

A plugin is a capability outside the kernel. It implements a typed port and can
live in `hive/plugins/` while it is local or experimental. A mature plugin can
move into its own repo without changing the kernel contract.

Plugin names use:

```text
<domain>_<kind>
```

Examples:

```text
confluence_connector
k8s_tool
```

## Port

A port is the stable contract between the kernel and the outside world. It says
what capability exists, not how it is deployed.

Current port kinds:

- `connector`;
- `tool`;
- `worker`;
- `channel`;
- `model`;
- `skill`.

## Data Plane And Control Plane

The kernel is the control plane. Plugins are the data plane.

The control plane coordinates and enforces rules. The data plane talks to real
systems, ingests data, performs bounded actions, and runs specialized work.

## Local-First

The system should run usefully on one machine. Cloud services and paid APIs are
optional escalations, not a base requirement.

## Cost Asymmetry

Cheap work should run often. Expensive models should run rarely and only when
they add value. This applies both to the product architecture and to the
development workflow.
