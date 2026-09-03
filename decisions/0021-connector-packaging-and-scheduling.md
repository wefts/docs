# ADR-21: Connector Packaging And Scheduling

## Status

Accepted (2026-09-03).

## Record Completeness

Complete. Design critic: Gemini, via `scripts/gemini_review.sh` from the
workspace root.

## Context

The Confluence and MediaWiki connectors currently have a typed kernel port but
no deployable runtime form. `hive/scripts/rebuild_graph.sh` loads both connector
source files with `mix run -r` from a sibling `hive/plugins/...` checkout. That
works only on an operator host with the source tree, `mise`, `mix`, and an
Elixir compiler installed.

The deployed Swarm image is a Mix release. It has the runtime needed to run the
kernel, not a compiler or plugin source. Therefore ingest does not run from the
same artifact that is deployed, and a Kubernetes migration would not have a
thing to schedule. The current form also leaves credentials, network reach, and
operator timing outside the deployment model.

The port boundary is sound: `Swarm.Ports.Connector.fetch/2` lets the kernel drive
pagination, preserve source ceiling reports such as `truncated: true`, maintain
the skip ledger, and send events through `Swarm.Ingest`. The missing decision is
how connector code becomes a runtime artifact and how that artifact is invoked.

This decision must preserve the workspace-level plugin boundary: the kernel must
not import plugin source as a hidden dependency, and the public `docs/`,
`swarm/`, and `hive/` repos must not contain secrets, intranet hostnames/IPs, or
private corpus content.

## Decision

Package each connector as its own runtime image and run finite pull ingestion as
a scheduled co-executed job: a kernel ingest runner plus the selected connector
image as a local sidecar, speaking the connector port over gRPC on localhost.

Concretely:

1. **Code delivery:** each connector gets a release-grade image built from its
   plugin source, with no dependency on a sibling checkout at runtime. The image
   contains the connector runtime and an entrypoint that exposes or invokes the
   connector protocol. The current `mix run -r ../../hive/plugins/...` loader is
   replaced by running connector images.
2. **Port boundary:** keep the kernel-driven pagination contract. The connector
   image exposes a gRPC `fetch` RPC equivalent to
   `Swarm.Ports.Connector.fetch/2`; the kernel-side ingest runner calls that
   RPC over localhost, drives cursors/pages, records skip-ledger entries,
   detects source ceilings, and ingests events through the existing
   `Swarm.Ingest` path.
3. **Credentials:** connector credentials are runtime configuration. On the
   current compose host they come from env files such as `hive/secrets.env`; in a
   cluster they come from Kubernetes Secrets. The connector image declares the
   required variables in its manifest and receives values through the scheduler,
   never from committed files and never from a source checkout.
4. **Scheduling:** finite pull ingestion runs as a scheduled job. In Kubernetes
   this is a CronJob whose Pod contains both the kernel ingest runner and one
   connector image as a sidecar. On Spark's current docker-compose host, use the
   same images under a host scheduler plus compose-run wrapper that starts the
   same pair. The scheduled unit completes when the kernel ingest runner
   finishes pulling all pages and exits non-zero on hard failure; orchestration
   then terminates the connector sidecar.
5. **Applicability:** Confluence and MediaWiki migrate identically. They are two
   `*_connector` plugins with the same artifact shape, manifest fields, secret
   injection pattern, scheduler pattern, and kernel-side pagination semantics.

The RPC protocol should be versioned independently from the connector image tag.
The manifest minimum from `docs/architecture/ports.md` still applies: stable
plugin name, port kind, runtime mode, entrypoint, protocol version, required
environment variables, declared capabilities, and side-effect safety class.

### What Speaks To What

The recommended topology is:

```text
scheduler
  starts co-executed job
    connector image
      gRPC server on localhost
    kernel ingest runner
      kernel-side connector client
        fetch RPC over localhost gRPC
          connector image
            external source
      Swarm.Ingest
```

This keeps the policy-heavy parts of ingestion in the kernel: pagination state,
event validation, origin/provenance handling, skip-ledger accounting, and
visibility-safe graph writes. It also keeps existing source-ceiling behavior
intact: when a connector reports a ceiling, the kernel observes and records
`truncated: true` instead of letting the connector silently cap results.

The transport is gRPC over localhost inside the scheduled job. In Kubernetes
that means same Pod. On Spark's compose host that means an ephemeral compose
definition or wrapper that puts the runner and connector on the same local
network namespace or private compose network. The main cost is that the
connector RPC must support cursor/page requests and structured result metadata
rather than a single "run sync" call. The benefit is that the current
correctness properties stay where they already exist.

The rejected alternative is connector-driven push:

```text
scheduler
  starts connector-only job
    connector drives pagination
    connector pushes events to ingest RPC
```

That shape makes each connector responsible for pagination semantics, skip
ledger behavior, retry boundaries, source-ceiling reporting, and partial-run
accounting. It is operationally attractive because the kernel only exposes an
ingest endpoint, but it duplicates correctness logic across connectors and makes
`truncated: true` and skip-ledger behavior easier to drift between Confluence and
MediaWiki.

Connector-driven push remains a possible future mode for sources that already
emit authoritative event streams. It should not be the migration path for the
current pull connectors.

### Running On Spark Tomorrow

Spark does not need Kubernetes before this design is useful. The same connector
images can run under the current docker-compose host:

- build or pull `confluence_connector` and `mediawiki_connector` images;
- pass credentials and source configuration from local env files at runtime;
- run the scheduled command through an operator-owned host timer plus a
  compose-run wrapper that starts the kernel ingest runner and connector sidecar
  together;
- connect to the same kernel/DB endpoints through configuration;
- require no mounted plugin source and no `mix` compiler on the runtime host.

When Kubernetes arrives, the job spec changes from "host timer starts a compose
run" to "CronJob starts the connector job." The artifact, manifest, protocol,
and secret names remain stable.

## Consequences

- Connectors become deployable artifacts rather than source files loaded from an
  operator checkout.
- The public kernel still depends only on a typed port and protocol version, not
  plugin source.
- The current deploy gap closes for both Confluence and MediaWiki: ingest can run
  without source checkout, without `mise`, without `mix`, and without an
  operator invoking a bespoke script.
- Scheduling and credentials move into deployment configuration, where compose
  and Kubernetes have equivalent concepts.
- The deployment model must orchestrate multi-container scheduled runs:
  Kubernetes sidecar Pods later, and ephemeral compose runs or a host-network
  equivalent on Spark now.
- The kernel gains a connector-client boundary and likely a small ingest-runner
  command if one does not already exist in release form.
- The hive repo gains image build definitions, manifests, env examples, and
  scheduler scaffold. It must keep real URLs, credentials, and corpus-specific
  values out of committed files.
- The connector protocol becomes a compatibility surface. Changes to request,
  page, event, warning, ceiling, skip, and error shapes need versioning and
  migration discipline.
- Out-of-process calls add serialization, network failure, and timeout handling.
  That is acceptable for scheduled pull ingestion, where external source latency
  already dominates and adapter failure is expected by the port model.

## Migration Cost

For each existing connector:

- add a plugin manifest that declares `connector` kind, out-of-process runtime
  mode, protocol version, entrypoint, required env vars, capabilities, and safety
  class;
- wrap the current `fetch/2` implementation behind the chosen RPC server inside
  a connector image;
- add a release/container build for the connector image;
- add local compose-run or host-scheduler scaffold that starts the kernel ingest
  runner and connector sidecar without a source checkout;
- add Kubernetes CronJob scaffold using the same images and env variable names;
- update the existing host script path so it invokes the packaged job or becomes
  an operator convenience wrapper rather than the canonical runtime path;
- prove that a full run preserves cursor progression, `truncated: true`
  reporting, skip-ledger output, origin/provenance behavior, and ingest event
  validation.

The two connectors should be migrated in one pattern pass, not as bespoke
services. The expected code cost is moderate: mostly packaging, protocol
adapter, scheduler scaffold, and release-command wiring. The risk is in
preserving ingestion semantics across the new boundary, not in the container
build itself.

## Gemini Critique

Gemini was run as the decorrelated critic through
`scripts/gemini_review.sh tmp/adr21-gemini-review-prompt.md`.

Verdict: **accept with changes**.

Gemini agreed that the proposal answered the five required questions and
preserved the important constraints, but found a lifecycle contradiction: the
draft said "connectors run as scheduled jobs" while also choosing a
kernel-driven pull model. If the connector alone is scheduled, it wakes up as an
idle RPC server and nothing tells the kernel to fetch. If the kernel ingest
runner is scheduled, the connector image must also be reachable during that run.

The critique recommended making the execution unit explicit: a scheduled
multi-container job with the kernel ingest runner as the driver and the
connector image as a sidecar, plus a concrete transport. This proposal adopts
that correction. I agree with Gemini's objection and verdict: finite pull
ingestion is the scheduled unit, the connector is a local sidecar, and the RPC
boundary is gRPC over localhost.

## Alternatives

- **Compile plugins into the kernel image.** Rejected. It is the shortest path to
  "something runs in Kubernetes," but every connector change would force a
  kernel release and the kernel would regain hidden plugin dependencies that the
  port architecture exists to prevent.
- **Keep `mix run -r` as the canonical runtime and mount source into jobs.**
  Rejected. It preserves the current failure mode: runtime depends on checkout
  layout, a compiler, and host tooling. It does not produce a deployable form.
- **Connector-driven push to a kernel ingest RPC.** Rejected for the current
  pull connectors. It is plausible for future streaming/event connectors, but it
  moves pagination correctness, skip accounting, and source-ceiling reporting out
  of the kernel and duplicates them per connector.
- **One shared connector host service containing all connectors.** Rejected for
  this stage. It reduces image count but couples connector release cadence and
  dependencies. Separate images match the port model and keep Confluence and
  MediaWiki independently replaceable while sharing the same packaging pattern.
