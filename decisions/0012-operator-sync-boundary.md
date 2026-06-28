# ADR-12: Operator Sync Boundary

## Status

Accepted

## Record Completeness

Complete

## Context

The local machine and Spark need the same `wefts` workspace shape, but remote
sync can overwrite production-adjacent files. It must be explicit operator work,
not an automatic agent default.

## Decision

Sync the local `wefts` workspace shape to the remote workspace root as:

```text
<remote workspace>/
  board/
  docs/
  swarm/
  hive/
  scripts/
```

`scripts/sync.sh` is local operator tooling and mirrors the workspace only when
called deliberately. It syncs root/operator entries and worktrees as separate
targets; it does **not** rsync git metadata. Claude/MCP config (`.claude/`,
`.mcp.json`) is **explicit-only** via the `agent-config` target, not part of
default `all` / `root` sync. Runtime state and secrets are excluded:

- `.git`
- `.env`
- `secrets.env`
- `*.local`
- `*.local.json`
- `.mcp.local.json`
- `.claude/settings.local.json`
- `data`
- `tmp`
- dependency/build/cache directories

## Consequences

- Spark receives the same workspace shape as local development.
- Push mirrors by default. Pull is additive by default; destructive pull requires
  an explicit `--delete`.
- Scratch and runtime data do not travel through rsync.
- Each GitHub-backed repo keeps its own remote history; `board/` keeps only local
  git history. The top-level workspace remains not a repo.

## Alternatives

- **Sync only `swarm/`** — rejected after the workspace split; it loses `docs/`,
  `board/`, `hive/`, and local operator context.
- **Automatic remote sync by agents** — rejected; remote sync is
  production-adjacent and belongs to the operator.
