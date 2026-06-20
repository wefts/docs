# Conventions

These conventions keep the workspace understandable as it grows.

## Repository Names

- `docs/` is the shared documentation canon.
- `swarm/` is the public kernel/control-plane repo.
- `hive/` is a private instance/deployment repo.
- `scripts/` is local operator tooling outside git.

## Plugin Names

Plugin directories and manifest names use:

```text
<domain>_<kind>
```

Examples:

```text
confluence_connector
k8s_tool
```

The domain comes first because people scan for the system or problem space
before they scan for the adapter type.

Valid kind names for now:

- `connector`;
- `tool`;
- `worker`;
- `channel`;
- `model`;
- `skill`.

## Scratch Space

Each project owns its own scratch directory:

```text
swarm/tmp/
hive/tmp/
```

Do not recreate a shared top-level `tmp/`.

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
