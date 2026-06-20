# Conventions

These conventions keep the workspace understandable as it grows.

## Repository Names And Ownership

Workspace identity and repository ownership are defined by
[ADR-10](../decisions/0010-wefts-workspace-split.md). Current status lives in
[../STATE.md](../STATE.md).

Do not duplicate the workspace layout here. This file records conventions that
sit below that decision.

## Plugin Names

Plugin naming is defined by the port contract in
[../architecture/ports.md](../architecture/ports.md). Do not duplicate the
allowed kind list here; this file only points at the authority.

## Scratch Space

Each repo owns its own scratch directory:

```text
swarm/tmp/
hive/tmp/
```

Do not recreate a shared top-level `tmp/`. Remote sync excludes scratch; see
[ADR-12](../decisions/0012-operator-sync-boundary.md).

## Documentation Placement

Put a document in `docs/` when it applies across repos.

Put a document in `swarm/docs/` when it is about the public kernel
implementation, kernel toolchain, or kernel runtime.

Put a document in `hive/` when it is about a concrete private instance,
deployment wiring, local plugins, secrets pointers, or data roots.

## Environment Variables

Use the `SWARM_` prefix for system-level variables that the kernel or plugins
consume. Instance-specific values live in `hive/.env` and examples live in
`hive/.env.example`.

Secrets are not stored in `.env.example`.
