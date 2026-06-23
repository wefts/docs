# ADR-8: Gate Thresholds

## Status

Accepted

## Record Completeness

Complete

## Context

The gate decides simple (handle locally) vs hard (escalate to the consilium), and
many other behaviors hang on numeric thresholds (bands, decay, saturation). Before
data exists there are no empirical thresholds — **gate cold-start** is a named open
problem (OP#2). The standing scar: a threshold set by intuition is wrong, and one
re-used eval set silently overfits selection.

## Decision

**No threshold is hardcoded by intuition.** Thresholds are derived from measured
distributions, and the eval harness (recall@k, MRR, nDCG, F1, regression CV) is
built **before** the feature it gates.

- **Frozen held-out split.** The gate sees the dev-eval set only; the held-out set
  is reserved for periodic audit and never tuned against.
- **Labels are externally produced and frozen** against the learning loop (no
  self-labeling into the gate).
- **Per-class gates** use multiple-comparison correction, an explicit delta
  tolerance, and a minimum sample size.
- A **dataset-refresh protocol** governs how fresh production outputs are labeled
  and stale labels replaced.
- **Cold-start.** Until calibrated, the gate uses **pass/fail decisions, not
  confidence** (ADR-3). Calibration uses isotonic/Platt regression on **50–200
  external labels** and requires ECE below threshold before confidence is trusted
  (ADR-9). Bands are **empirical and embedding-model-scale-specific**, so they are
  re-derived whenever the embedding model changes (ADR-6).

Tuning parameters (`lambda`, `S`, `mu`, bands) live in **one tuning inventory**, not
scattered as literals.

## Consequences

- Cold-start thresholds are provisional and must be revisited once real escalation
  data exists; OP#2 (how the *first* labels are produced/bootstrapped) stays open.
- Bands are not portable across embedding models — an embed-model swap forces
  re-derivation (the recurring scar this ADR exists to stop).
- The held-out reserve costs labeled data up front but is the only defense against
  selection overfitting across cycles.

## Alternatives

- **Hardcoded / intuited thresholds.** Rejected — unmeasurable; "did this help?"
  becomes unanswerable.
- **One eval set reused every cycle.** Rejected — overfits selection; hence the
  frozen held-out split.
- **Trusting uncalibrated confidence scores.** Rejected — confidence is a heuristic
  score until isotonic/Platt + ECE; pass/fail until then.
