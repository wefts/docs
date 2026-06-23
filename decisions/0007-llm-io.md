# ADR-7: LLM I/O

## Status

Accepted

## Record Completeness

Complete

## Context

Large models are rare, deliberate escalations (gate + consilium). Their input and
output need a stable, typed contract so disagreement can be captured rather than
flattened — and so a fluent-but-wrong model answer cannot quietly drive an action.

## Decision

The LLM I/O contract is **structured output + fencing + fail-loud +
verify-then-climb**:

- Model decisions use **structured output or token anchors, never substring
  parsing**. Code owns lists, formatting, and identifiers — not the model.
- **Untrusted external text enters prompts fenced as data**, and outputs are
  **validated before any action** is taken on them.
- **Failures are fail-loud, never success-shaped** — a model that cannot answer
  must produce a recognizable failure, not a confident-looking guess.
- Escalation uses **verify-then-climb**: try cheap, verify the result, climb to a
  more expensive model only when verification fails.

**Caveats (limits this contract does not remove):**

- **Shape is not content.** Structured output validates *form* only. Any model
  field that becomes an external-action parameter must be **reauthorized against a
  code-owned allowlist** — a well-formed value is not an authorized one.
- **Confident-wrong** remains a limit of verify-then-climb. Deterministic checks
  catch only *self-declared* inability; fluent wrong answers need a
  **different-model-family judge** and a **measured judge-accuracy metric**.

The **cost/budget dimension** of LLM I/O (per-call ceilings, ground-before-model,
hierarchical token buckets, circuit breakers) is finalized in **T5**
(`board/` roadmap) — this ADR fixes the I/O *contract*; T5 adds the *budget*.

## Consequences

- Model disagreement is kept as signal, not discarded (ties to ADR-3 and to
  `../standards/verification.md`); the consilium aggregates structured outputs
  rather than flattening them to one string.
- Prompt-injection via external text is bounded: external text is data, and
  outputs are re-authorized before acting.
- Detecting confident-wrong answers requires a cross-family judge — a measured
  cost the design accepts rather than pretends away.

## Alternatives

- **Substring / regex parsing of model output.** Rejected — brittle and
  injection-prone; the model ends up owning identifiers it should not.
- **Trusting structured output as authorization.** Rejected — shape is not
  content; validated form is not an allowlisted action.
- **Silent or success-shaped failure.** Rejected — hides inability and lets a
  guess flow downstream as if it were an answer.
