# ADR-12: Operator Sync Boundary

## Status

Accepted

## Context

The local machine and Spark need the same Wefts workspace shape, but remote sync
can overwrite production-adjacent files. It must be explicit operator work, not
an automatic agent default.

## Decision

Sync the whole local Wefts workspace to Spark as:

```text
~/Swarm/
  docs/
  swarm/
  hive/
  scripts/
```

`scripts/sync.sh` is local operator tooling and mirrors the workspace only when
called deliberately. Runtime state and secrets are excluded:

- `.env`
- `secrets.env`
- `*.local`
- `data`
- `tmp`
- dependency/build/cache directories

## Consequences

- Spark receives the same workspace shape as local development.
- Sync can still use `--delete`, but only under explicit human control.
- Scratch and runtime data do not travel through rsync.
- Each repo keeps its own git history; the top-level workspace remains not a
  repo.

## Alternatives

- **Sync only `swarm/`** — rejected after the workspace split; it loses `docs/`,
  `hive/`, and local operator context.
- **Automatic remote sync by agents** — rejected; remote sync is
  production-adjacent and belongs to the operator.
