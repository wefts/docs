# ADR-5: Topology & Visibility

## Status

Accepted

## Context

Who-can-see-what in the shared graph is **security-critical**: the visibility filter
is the privacy boundary, and it must hold under load. Topology (how nodes/edges are
grouped and scoped) and visibility are one decision.

## Decision

> TODO: transplant the actual topology + visibility-filter decision from
> swarm_architecture_spec.md. Record the scoping model and the filter's enforcement
> point.

## Consequences

- The visibility-filter **threat model under load** is a named open problem; privacy
  here is a security property, not a preference.

## Alternatives

> TODO: transplant rejected visibility models.
