# How To Write An ADR

This workspace does not require central ADR files for every decision. Use an ADR
only when a decision is durable, architectural, and expensive to reverse.

Repo-specific ADRs should live in the repo that owns the decision.

## When To Write One

Write an ADR when the decision changes one of these:

- public contracts;
- repository boundaries;
- storage substrate;
- plugin ABI;
- security or permission model;
- deployment topology;
- long-term technology choice.

Do not write an ADR for routine implementation detail.

## Shape

Use this structure:

```text
# ADR-N: Title

## Status

Proposed | Accepted | Superseded

## Context

What problem forced this decision?

## Decision

What did we choose?

## Consequences

What becomes easier, harder, or impossible?

## Alternatives

What did we reject and why?
```

## Rules

- One ADR records one decision.
- State the tradeoff, not only the chosen path.
- Link to measurements or spikes when they exist.
- Do not silently edit accepted ADRs to mean something else.
- Supersede old decisions with a new decision when the architecture changes.
