# Ports and Plugin Contracts

Swarm grows by adding adapters behind typed ports, not by teaching the kernel
about every external system. A port is the stable capability contract; an adapter
is one implementation of that contract.

This is the rule that lets a public kernel coexist with private, experimental,
or third-party integrations.

> **Ports protect the *call* boundary, not the *data* boundary.** A plugin can
> obey every port signature and still write malformed or scope-leaking rows into
> the shared graph. That second boundary is a separate contract — the graph schema
> is write-validated (type/scope vocabulary, the ADR-5 visibility invariant,
> reliability, provenance shape) and stamped with a schema version, enforced at
> `Swarm.Graph.Store`. See swarm ADR-4
> (`../../swarm/docs/decisions/0004-graph-integrity-contract.md`).

### Connector evidential-origin contract (ADR-13)

A `connector` event carries two distinct keys, and the distinction is a
**correctness invariant**, not a convenience:

- **`provenance`** — the *emission instance*: one per event/fetch. It dedups
  re-delivery (an event must never count twice).
- **`origin`** — the *evidential source identity*: derived from **source/content
  identity**, **stable across re-emissions** of the same fact. Reinforcement and
  corroboration count **distinct origins**
  ([ADR-13](../decisions/0013-evidential-origin.md)), so re-emitting or
  re-deriving the same fact MUST reuse the same `origin`, and a genuinely
  independent source MUST get a new one.

A **derivative-capable** connector (one whose source re-publishes, mirrors, or
syndicates content — most real sources) MUST set `origin` from the upstream
source identity. Omitting it defaults `origin := provenance` (every event its own
origin), which is correct only where each event is genuinely independent; for a
derivative-capable source that default **re-opens** the correlated-evidence
hazard ADR-13 closes. The kernel counts and caps by `origin`; determining it is
the connector's job, at the boundary where source identity is known.

## Port Kinds

The current top-level kinds are:

- `connector` ingests or syncs data from an external system.
  Example: `confluence_connector`.
- `tool` performs a bounded external action.
  Example: `k8s_tool`.
- `worker` runs background analysis, consolidation, or maintenance.
  Example: `memory_worker`.
- `channel` exposes an interaction surface.
  Example: `cli_channel`.
- `model` provides model, embedding, reranking, or inference calls.
  Example: `ollama_model`.
- `skill` packages task-specific behavior and prompts.
  Example: `incident_skill`.

These names are deliberately generic. Specific systems belong in the domain
part of the plugin name, not in the kernel.

## Naming Rule

Plugin manifests and plugin directories should use:

```text
<domain>_<kind>
```

Examples:

```text
hive/plugins/
  confluence_connector/
  k8s_tool/
```

The `domain` answers "what system or problem space is this for?" The `kind`
answers "what contract does it implement?"

Avoid reversed names such as `connector_confluence`: they group by technology
instead of by domain and become harder to scan once the hive grows.

## Manifest Minimum

The exact manifest schema can evolve, but every plugin manifest should identify:

- stable plugin name;
- port kind;
- runtime mode;
- entrypoint;
- protocol version;
- required environment variables;
- declared capabilities;
- safety class for side effects.

The manifest is the bridge between the private hive and the public kernel. It is
also the future migration path from "plugin lives inside `hive/plugins/`" to
"plugin lives in its own repo."

## Runtime Modes

Adapters can be loaded in several ways, depending on maturity and risk:

- In-process dev adapter:
  fast local experiments with trusted code; same VM/process.
- Out-of-process adapter:
  the default serious plugin shape; gRPC or another typed IPC.
- Sidecar/container:
  plugin has its own deps or language runtime; Compose/Kubernetes service.
- Remote service:
  capability already exists elsewhere; network API.

The kernel should treat these as deployment choices behind the same port. A
plugin should not require kernel code changes just because it moves from
`hive/plugins/` to a sidecar or a remote service.

## Third-Party Tools

A local tool such as `<local k8s controller tool>` should not be added by
importing its source tree into the kernel. It should be wrapped as one of:

- `k8s_tool` if it performs bounded Kubernetes actions;
- `k8s_connector` if it primarily watches and ingests cluster state;
- `k8s_worker` if it runs a long-lived control or reconciliation loop.

The wrapper can start as a thin adapter in `hive/plugins/k8s_tool/` and later move
to a separate repo if it becomes reusable.

## Kernel Expectations

The kernel may assume:

- a plugin has a declared kind and protocol version;
- calls cross a typed boundary;
- side effects are explicit and auditable;
- secrets arrive through the hive environment, never through committed files;
- adapter failure is normal and must not bring down the kernel.

The kernel must not assume:

- plugin source is present inside `swarm/`;
- plugins share the kernel's language or dependency graph;
- plugin data is safe to commit;
- a local path exists on another machine.
