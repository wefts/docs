# ADR-9: Stigmergic-Loop Stability

## Status

Accepted

## Record Completeness

Complete

## Context

Stigmergic feedback (workers reacting to traces they and others leave) can
reinforce itself into divergence or premature convergence. Formal convergence —
not just rate limits — is a named open problem.

## Decision

Stability rests on a **saturating trace with dominant decay**, not on rate limits
alone:

- Trace strength saturates rather than growing linearly:
  `strength = f(seen_count) * exp(-lambda * age)`, with the Hill form
  `f(n) = log(1+n) / (log(1+n) + S)` — monotone in `n`, bounded in `[0, 1)`.
- **Decay is the dominant pole.** A one-off reinforcement burst fades; only
  sustained independent evidence holds an edge up.
- **Reinforcement comes only from independent ingest events, not internal
  re-detections.** This is a precondition for ADR-3's confidence aggregation —
  without it, the system confirms its own beliefs.
- Tuning parameters `lambda`, `S`, `mu` live in **one tuning inventory** (ADR-8),
  never as scattered literals.
- **Cold-start calibration** uses isotonic/Platt regression on 50–200 external
  labels and requires ECE below threshold; until then, confidence is not trusted
  (shared with ADR-8).

In the kernel this is `Swarm.Graph.Strength` (pure functions; `lambda`/`S` from
config). What it gives: bounded, decaying traces that cannot run away from a single
source. What it does **not** give: a *formal* convergence guarantee, and it does not
distinguish correlated-but-provenance-distinct events (see Consequences).

## Consequences

- Current stability rests on rate limiting; **formal convergence is unproven** and
  remains open. Interacts with decay-vs-reinforcement tuning (lambda) and with
  cortical-voting-style premature convergence (glossary §4).
- **Reinforcement treats provenance-distinct events as independent — they are
  not.** `seen_count` increments once per distinct `(edge, provenance)` event
  (the `edge_provenance` guard in `Swarm.Graph.Store`). That guard is exact for
  one hazard — a *single* event must not count twice, which closes the endogenous
  confirmation loop — but silent on another: *distinct yet correlated* events. N
  derivatives of one source reinforce an edge N times, as if N independent
  witnesses had confirmed it. This is the same shared-ancestor overcounting that
  ADR-3 corrects in the **confidence** dimension, reappearing here in the
  **strength** dimension, where it is today only *mitigated* (Hill saturation
  bounds it; decay dominates a one-off burst), not *corrected*. Residual hazard:
  a connector emitting provenance-fresh but evidentially-stale events refreshes
  `last_seen` indefinitely and pins an edge near saturation, defeating decay —
  turning a decay-dominated edge effectively immortal with no new evidence behind
  it. Full analysis and solution space:
  [confidence-calculus.md → Independence in the strength dimension](../architecture/confidence-calculus.md#independence-in-the-strength-dimension).
  Open until a decision is taken among: connector provenance contract /
  per-source reinforcement cap / lineage-aware reinforcement.

## Alternatives

- **Linear `f(seen_count)`.** Rejected — positive feedback without saturation,
  against a dominant decay, oscillates or runs away; linear growth makes edges
  effectively immortal.
- **Reinforcement from any re-detection (including internal).** Rejected — closes
  an endogenous confirmation loop and breaks the independence precondition ADR-3
  relies on; reinforcement must come from independent ingest events only.
- **Rate limiting as the primary control.** Rejected as *sufficient* — it caps the
  rate of change but not the fixed point; saturation + decay bound the steady
  state. Rate limits may still complement it operationally.
