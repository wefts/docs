# Current State

This file records the current shape of the Swarm workspace. It is not a product
roadmap and not a task list. It is the shortest answer to: "what is true right
now?"

## Workspace

The top-level directory is a workspace, not a repo:

```text
swarm/
  docs/       shared architecture, standards, vocabulary
  swarm/      public kernel/control-plane repo
  hive/       private instance/deployment repo
  scripts/    local operator scripts, outside git
  .mcp.json   local agent/tool wiring
```

`docs/`, `swarm/`, and `hive/` may be separate git repos. The top-level
workspace is not the unit of publication.

## Canonical Split

`docs/` is the shared canon:

- cross-repo architecture;
- standards and guardrails;
- shared vocabulary;
- current workspace state.

`swarm/` is the public kernel repo:

- Elixir/OTP kernel;
- Python ML service and CLI channel;
- Protobuf contracts;
- kernel infra substrate;
- kernel-specific architecture and implementation docs.

`hive/` is the private instance repo:

- deployment wiring;
- environment examples and secrets pointers;
- enabled plugins;
- private data roots;
- instance-specific notes.

`scripts/` is local operator tooling. It is intentionally outside git and is not
part of the public product.

## Current Architecture Direction

The project is a local-first heterogeneous cognitive swarm:

- cheap specialized processes run continuously;
- expensive models are rare escalations;
- coordination happens through a shared graph;
- the public kernel stays small;
- concrete integrations live outside the kernel behind typed ports.

The shipping model is a small polyrepo workspace:

- public `swarm/` for stable kernel code;
- private `hive/` for local deployment and early plugins;
- optional standalone plugin repos later;
- shared `docs/` for rules that apply across repos.

## Current Boundaries

- Repo-specific details stay in the owning repo.
- Private systems, private data, and secrets do not move into `docs/`.
- The top-level docs may point to `swarm/docs/` or `hive/README.md`, but should
  not duplicate detailed implementation docs.
- Each project owns its own `tmp/`.
- Remote sync is a human/operator action.

## Open Documentation Gaps

- Some shared standards are intentionally compact and will grow when the project
  has more code and plugins.
- Kernel-specific docs still live in `swarm/docs/`, by design.
- Hive-specific operational details still live in `hive/`, by design.
