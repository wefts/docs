# Concepts

This is the short conceptual model. The longer working vocabulary lives in
[glossary.md](glossary.md).

## Swarm

A swarm is a system made from many small, specialized processes. Each process is
limited, but the system becomes useful because the processes share memory,
observe each other's traces, and escalate only when necessary.

## wefts

`wefts` is the workspace and GitHub namespace: the weave that ties `docs/`,
`swarm/`, and `hive/` together. It is not a deployable product and not a single
repo.

When names are ambiguous, use:

- `wefts` for the workspace/namespace;
- `Swarm` for the product/system;
- `swarm/` for the public kernel repo;
- `Hive` for a deployment environment;
- `hive/` for a public deployment scaffold repo.

## Hive

A hive is a concrete Swarm instance. It contains deployment wiring, env examples,
enabled plugins, data roots, and integration choices.

Committed hive files are public scaffold. Private values stay in gitignored files,
volumes, or operator config. The public kernel is not allowed to know concrete
deployment details.

## Kernel

The kernel is the stable control plane. It owns scheduling, supervision, typed
ports, storage abstractions, guardrails, and the core runtime contracts.

The kernel should be boring, small, and public.

## Plugin

A plugin is a capability outside the kernel. It implements a typed port and can
live in `hive/plugins/` while it is local or experimental. A mature plugin can
move into its own repo without changing the kernel contract.

Plugin naming and allowed port kinds are defined in
[../architecture/ports.md](../architecture/ports.md).

## Port

A port is the stable contract between the kernel and the outside world. It says
what capability exists, not how it is deployed.

The authoritative port-kind list lives in
[../architecture/ports.md](../architecture/ports.md).

## Data Plane And Control Plane

The kernel is the control plane. Plugins are the data plane.

The control plane coordinates and enforces rules. The data plane talks to real
systems, ingests data, performs bounded actions, and runs specialized work.

## Project

A Project is the user-facing data-access and sharing container. Projects own
Sources/Connectors; membership in a Project gives an actor the source scopes produced by
that Project.

Project access is defined in [../architecture/access-model.md](../architecture/access-model.md).

## Source Scope

A Source scope is the graph visibility coordinate behind one concrete source instance, for
example `src:<source_uuid>`. Human labels such as `wiki` or `confluence` are labels, not
security keys.

## Groups

The fixed workspace groups are `Wheel`, `Admins`, and `Staff`.

Groups do not grant source visibility directly. They can be Project members, and that
Project membership gives their users access to the Project's source scopes. Roles confer
capabilities, not data visibility.

## Local-First

The system should run usefully on one machine. Cloud services and paid APIs are
optional escalations, not a base requirement.

## Cost Asymmetry

Cheap work should run often. Expensive models should run rarely and only when
they add value. This applies both to the product architecture and to the
development workflow.
