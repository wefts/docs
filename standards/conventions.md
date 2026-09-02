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

Put a document in `hive/` when it is about public deployment scaffold, local
plugins, env structure, secrets pointers, or data roots. Private values stay in
gitignored files, volumes, or operator config.

## Environment Variables

Use the `SWARM_` prefix for system-level variables that the kernel or plugins
consume. Instance-specific values live in `hive/.env` and examples live in
`hive/.env.example`.

Secrets are not stored in `.env.example`.

## Commit Messages

**Every commit uses [Conventional Commits](https://www.conventionalcommits.org/):
`type(scope): summary`.** This is not style — the release tooling reads it.

`swarm/scripts/release.sh` runs `git cliff --bump`, which derives the next version from
commit types: a `feat` is a minor bump, a `fix` is a patch, and anything unrecognised is
neither. A non-conventional message therefore **understates the release**. This has already
happened: two commits adding deterministic network semantics and the whole calibration loop
were written as `Add …`, `--bump` computed `v0.3.1` instead of `v0.4.0`, and both features
were filed under "Other" in the changelog. The messages had to be rewritten before tagging.

Types in use: `feat`, `fix`, `docs`, `perf`, `refactor`, `style`, `test`, `chore`, `revert`.
Scope is the subsystem — `core`, `graph`, `enrichment`, `world-map`, `calibration`,
`deploy`, `scripts`, `board`. Breaking changes take `!` before the colon.

`swarm/cliff.toml` keeps `filter_unconventional = false` and a catch-all parser, so a
non-conventional commit is *visible* rather than silently dropped — that is a safety net for
old history, not permission to skip the convention.

Applies to agents and humans alike, in every repo. Rewriting a message is cheap while a
branch is unpushed and expensive afterwards, so get it right at commit time.
