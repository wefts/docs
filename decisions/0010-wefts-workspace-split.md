# ADR-10: `wefts` Workspace Split

## Status

Accepted

## Record Completeness

Complete

## Context

The project outgrew a single-repo mental model. The public product kernel,
private deployment state, shared docs, and local operator scripts have different
audiences and different publication rules.

## Decision

Use **`wefts`** as the workspace and GitHub namespace. Keep three names
separate:

- **`wefts`** — the workspace/namespace that ties the repos together.
- **Swarm** — the product and public kernel/control-plane repo.
- **Hive** — a concrete private deployment environment.

The top-level workspace is not a git repo. The owned units are:

```text
wefts/
  docs/      public shared canon
  swarm/     public product kernel
  hive/      private deployment instance
  scripts/   local operator tooling, outside git
```

## Consequences

- Cross-repo architecture and standards live in `docs/`.
- Kernel-specific implementation detail lives in `swarm/`.
- Private deployment detail lives in `hive/`.
- Local sync/operator scripts stay outside product repos.
- Shared docs use `wefts` / Swarm / Hive as distinct terms; repo-specific docs
  may still need spot checks.

## Alternatives

- **One monorepo** — rejected for now; it mixes public kernel code with private
  deployment state.
- **Many plugin repos immediately** — rejected for now; too much coordination
  overhead before the plugin ABI is stable.
