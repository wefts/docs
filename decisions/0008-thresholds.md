# ADR-8: Gate Thresholds

## Status

Accepted

## Context

The gate decides simple (handle locally) vs hard (escalate to the consilium). Before
data exists there are no empirical thresholds — **gate cold-start** is a named open
problem.

## Decision

> TODO: transplant the threshold model and initial values from
> swarm_architecture_spec.md. Record the cold-start strategy (defaults before data)
> and how thresholds adapt once signal accumulates.

## Consequences

- Cold-start thresholds are provisional and must be revisited once real escalation
  data exists.

## Alternatives

> TODO: transplant rejected gating strategies.
