# Shared Documentation

This repository is the shared documentation layer for the `wefts` namespace /
workspace. It describes the architecture, standards, and vocabulary that apply
across the public Swarm kernel repo, the private Hive repo, and the local
operator scripts.

Repo-specific instructions stay in the repo they belong to:

- `swarm/` documents the public kernel, ports, infra substrate, and runtime code.
- `hive/` documents the private deployment instance, enabled plugins, env files,
  secrets pointers, and data roots.
- `scripts/` is local operator tooling and is intentionally outside git.

## Start Here

- [STATE.md](STATE.md) — current workspace state, ownership, and what is
  considered canonical today.
- [architecture/overview.md](architecture/overview.md) — workspace shape,
  ownership boundaries, and the shipping model.
- [architecture/ports.md](architecture/ports.md) — typed extension points and
  plugin naming rules.
- [reference/concepts.md](reference/concepts.md) — the short conceptual model.
- [standards/guardrails.md](standards/guardrails.md) — what the agent may do,
  must ask about, and must never do.
- [standards/workflow.md](standards/workflow.md) — the cross-repo working loop.
- [standards/conventions.md](standards/conventions.md) — naming, ownership, and
  documentation conventions.
- [standards/code-style.md](standards/code-style.md) — cross-repo code style.
- [standards/verification.md](standards/verification.md) — how work is checked
  before it is trusted.
- [reference/glossary.md](reference/glossary.md) — working vocabulary and known
  architecture terms.

## Documentation Boundary

Keep this layer general. It should answer "what is this system and what rules
hold across all repos?" It should not name private credentials, real customer
systems, local-only paths, or one-off deployment details.

When in doubt:

- Put cross-repo architecture and standards here.
- Put kernel implementation details in `swarm/docs/`.
- Put private instance/deployment details in `hive/`.
- Put temporary notes in each repo's own `tmp/`.
