# ADR-10: Wefts Workspace Split

## Status

Accepted

## Context

The project outgrew a single-repo mental model. The public product kernel,
private deployment state, shared docs, and local operator scripts have different
audiences and different publication rules.

## Decision

Use **Wefts** as the workspace and GitHub org identity. Keep three names
separate:

- **Wefts** — the workspace/org that ties the repos together.
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
- Older docs that say "Swarm workspace" need a naming pass.

## Alternatives

- **One monorepo** — rejected for now; it mixes public kernel code with private
  deployment state.
- **Many plugin repos immediately** — rejected for now; too much coordination
  overhead before the plugin ABI is stable.
