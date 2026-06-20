# ADR-1: Concurrency

## Status

Accepted

## Context

Concurrency and process supervision are hard to retrofit. They must be a
property of the kernel from day one, not bolted on later.

## Decision

Logic and coordination run on **Elixir/OTP** (supervision trees, "let it crash").
Concurrency **invariants are baked into the kernel skeleton** from the first build
slice.

> TODO: transplant the specific concurrency invariants from
> swarm_architecture_spec.md (what exactly is guaranteed, and where it is enforced).

## Consequences

- Supervision, restarts, and back-pressure come from a mature runtime.
- The kernel's hard core is Elixir; polyglot work lives outside it (see ADR-6,
  ADR-7).

## Alternatives

> TODO: transplant rejected concurrency models and why.
