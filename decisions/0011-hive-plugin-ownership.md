# ADR-11: Hive Plugin Ownership

## Status

Accepted

## Record Completeness

Complete

## Context

Early integrations need to evolve near the deployment environment they depend
on. At the same time, the public kernel must not import or vendor concrete
third-party tools, private connectors, corpora, or secrets.

## Decision

Early plugins live in the private `hive/plugins/` tree. Mature reusable plugins
may move to their own repos later, but the kernel contract must not change when
that happens.

Plugin directory names, manifest names, and allowed port kinds are defined by
the port contract in [../architecture/ports.md](../architecture/ports.md).
ADR-11 owns *where early plugins live*; `ports.md` owns *what plugin kinds and
names mean*.

## Consequences

- The public kernel depends on typed ports and manifests, not plugin source.
- Hive may contain private or experimental adapters.
- A plugin can graduate from `hive/plugins/` to a standalone repo without a
  kernel rewrite.
- Plugin names group by domain first, which keeps a growing hive easy to scan.

## Alternatives

- **Put plugins in `swarm/`** — rejected; it expands the public support surface
  and risks pulling private state into the kernel repo.
- **Require standalone plugin repos from day one** — rejected; too much ceremony
  before the ABI is stable.
