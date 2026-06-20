# Workflow

This is the cross-repo workflow. Repo-specific commands live in the owning repo.

## Before Editing

1. Check which repo owns the change.
2. Read the nearby docs and code.
3. Check git status in the repo being edited.
4. Keep unrelated dirty work intact.

## Ownership

Use this routing rule:

- cross-repo rule or vocabulary change: `docs/`;
- kernel/runtime/protocol change: `swarm/`;
- private deployment/plugin/env change: `hive/`;
- local sync or operator helper: `scripts/`.

## Change Shape

Prefer small, coherent changes that can be reviewed in one repo at a time.
Cross-repo changes are fine when the boundary itself changes, but the reason
should be explicit.

## Verification

Run the cheapest relevant checks first:

- Markdown docs: `markdownlint` on touched files.
- Shell scripts: `bash -n`.
- Kernel code: the checks documented in `swarm/`.
- Hive deployment: `docker compose config` when Docker is available.

If a tool is not available in the current shell, record that honestly instead of
pretending the check ran.

## Remote Sync

Remote sync is not part of the default agent workflow. It is a deliberate
operator action because it reaches production-adjacent state.

## Finishing

Before calling work done:

- verify the changed files;
- report which checks passed;
- report which checks could not run;
- keep repo-specific changes in their owning repo.
