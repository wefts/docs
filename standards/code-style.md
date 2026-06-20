# Code Style

These are cross-repo rules. Language-specific details belong in the owning repo,
for example `swarm/docs/guides/` for the kernel.

## Defaults

- Prefer clear, boring code over clever code.
- Keep side effects at boundaries.
- Make errors explicit and typed where the language allows it.
- Read configuration at runtime boundaries, not at import time.
- Keep secrets out of code, docs, logs, and committed env files.
- Use stable names for public contracts.
- Avoid speculative abstractions until duplication or complexity is real.

## Boundaries

Code must respect the workspace split:

- `swarm/` code must not import concrete plugin source from `hive/`.
- `hive/` may depend on `swarm/` contracts.
- `docs/` must not contain executable product logic.
- `scripts/` may automate local operator work, but should not become hidden
  product code.

## Comments

Use comments to explain why something exists or why an unusual choice is safe.
Do not comment what the next line already says.

## Formatting

Use each repo's formatter and linter. If a repo has no formatter yet, keep
Markdown and shell files readable, wrapped, and consistent with neighboring
files.

## Public Contracts

Treat these as compatibility surfaces:

- Protobuf schemas;
- plugin manifests;
- environment variable names;
- port names and kind names;
- documented directory layout.

Changing a public contract requires a deliberate migration note in the owning
repo.
