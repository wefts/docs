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

**Budget (cost ceiling), finalized in T5.** Cost asymmetry is a principle; this
makes it a *ceiling*. The scar: glpi-agent sent 385k input tokens in one turn
because a tool dumped a raw payload into a model call.

- **Two-layer ceiling, config-driven.** The **global backstop** is enforced at the
  model boundary itself (`Swarm.ML.Generation.generate/3`): *any* caller — present
  or future, consilium or not — is **refused fail-loud** (`{:error, {:over_budget,
  estimated, ceiling}}`) before the RPC ship-out, so no model call can exceed it.
  The **consilium** adds a tighter, earlier per-escalation ceiling so an
  escalation is refused before the panel even runs. Either way: refuse, **never
  silently truncate**.
- **Per-call, so panel fan-out multiplies it.** The ceiling bounds one prompt; an
  N-model panel spends ≈ N × (panel prompt) + judge. The bound is per-call by
  design (each call is what hits a model); total escalation cost is therefore
  bounded by `panel-width × ceiling`, and is what telemetry accounts.
- **Ground/compress before the model, never pass a raw source/tool payload.** The
  kernel *rejects* an ungrounded payload; producing a bounded, grounded context is
  the caller's job upstream (the digest lesson, 323k→20k). The estimate (bytes/4)
  is a structural proxy, not a tokenizer — ample against a catastrophic dump,
  approximate near the boundary.
- **Cost accounting** — estimated tokens in/out for the *whole* escalation (panel
  fan-out + judge), plus a telemetry event on **refusal**, so both a cost
  regression and a spike of over-budget attempts are observable.

Mechanism + numbers: `swarm/docs/design/llm-budget.md`; enforced in
`Swarm.LLM.Budget`, the `Swarm.ML.Generation` boundary, and the `Swarm.Consilium`
escalation path.

## Consequences

- Model disagreement is kept as signal, not discarded (ties to ADR-3 and to
  `../standards/verification.md`); the consilium aggregates structured outputs
  rather than flattening them to one string.
- Prompt-injection via external text is bounded: external text is data, and
  outputs are re-authorized before acting.
- Detecting confident-wrong answers requires a cross-family judge — a measured
  cost the design accepts rather than pretends away.
- A single escalation now has a hard cost ceiling: the catastrophic-payload path
  fails loud at the boundary instead of silently spending. The trade-off is that
  an over-ceiling query is *refused* rather than auto-compressed — grounding is
  pushed upstream, which is where the source/tool context actually lives.

## Alternatives

- **Substring / regex parsing of model output.** Rejected — brittle and
  injection-prone; the model ends up owning identifiers it should not.
- **Trusting structured output as authorization.** Rejected — shape is not
  content; validated form is not an allowlisted action.
- **Silent or success-shaped failure.** Rejected — hides inability and lets a
  guess flow downstream as if it were an answer.
