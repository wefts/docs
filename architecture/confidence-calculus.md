# Confidence Calculus

How a conclusion reached by traversing the graph gets a single trust number, when
every edge is only partially reliable. This is normative. The chosen algebra
below is a locked decision — see
[ADR-3: Confidence Calculus](../decisions/0003-confidence-calculus.md).

## The problem

Traversal reaches a conclusion through one or more *paths* of typed edges. Each
edge carries a reliability `r ∈ (0, 1]` — how much the link is to be trusted. We
need to collapse all of that into one number: how much to trust the conclusion.

There are exactly two combinators, and one trap:

- combine reliabilities **along** a path (the AND direction);
- combine confidence **across** paths that reach the same conclusion (the OR
  direction);
- the trap: paths that look independent but share a source, which silently
  **double-counts** the same evidence and inflates confidence.

Everything below exists to get those three things right and in one coherent
algebra.

## AND — confidence along a path

A path is a chain of edges. The conclusion holds only if every link holds, so the
path's confidence is the **product** of its edge reliabilities:

```text
C(path) = r₁ · r₂ · … · rₙ
```

Computed in **log-space** as a sum, both for numerical stability and because it
makes the penalty on depth explicit:

```text
log C(path) = Σ log rᵢ      (each log rᵢ ≤ 0)
```

Every extra edge multiplies by something `≤ 1`, i.e. adds a non-positive term in
log-space. Longer chains are strictly less trusted than shorter ones, and a single
weak link drags the whole path down. This is the correct behaviour: a conclusion
that depends on a long fragile chain of inferences *should* be held weakly.

## OR — confidence across independent paths

When several **independent** paths reach the same conclusion, they corroborate.
The combinator is **noisy-OR**: the conclusion fails only if every path
independently fails to support it.

```text
C = 1 − Π (1 − Cⱼ)      over paths j
```

More corroborating paths push confidence up, asymptotically toward (but never
reaching) 1. Two mediocre independent paths can add up to a strong conclusion —
which is exactly what corroboration should mean.

This formula is **only valid when the paths are independent.** That condition is
not a footnote; it is the whole difficulty.

## The independence trap — shared ancestors

If two paths share a common source, they are not two pieces of evidence — they are
one piece of evidence seen twice. Noisy-OR over them counts that source's support
twice and reports false confidence.

The rule:

- **Within** a group of paths that share an ancestor, take the **max** — the
  single strongest path stands for the group's (shared) evidence.
- **Across** independent groups, take the **noisy-OR** of those per-group maxima.

Taking the max within a shared-ancestor group is the conservative, correct move:
it refuses to let the same source vote more than once, and keeps the strongest
view of that source's evidence rather than averaging it away.

This is not a novel problem. It is classic **overcounting / double-counting** in
loopy inference [Weiss 2000; Ihler et al. 2005]. Naming it correctly matters,
because it points at a real solution space (below) instead of a home-grown
heuristic.

## The algorithm, assembled

1. For each path, compute `C(path)` as the log-space product of its edge
   reliabilities (AND).
2. Partition the paths into **independence groups**: paths sharing an ancestor go
   in the same group.
3. Within each group, take the **max**.
4. Across groups, take the **noisy-OR** of the per-group maxima.

### Worked example

Conclusion `X`, supported by three paths:

```text
A:  S ──0.9──► m ──0.8──► X     C(A) = 0.72
B:  S ──0.7──► n ──0.9──► X     C(B) = 0.63     (shares source S with A)
C:  T ──0.85──────────────► X   C(C) = 0.85     (independent source T)
```

Group `{A, B}` shares `S` → take max = **0.72**. Group `{C}` → **0.85**.

```text
C(X) = 1 − (1 − 0.72)(1 − 0.85) = 1 − 0.28 · 0.15 = 0.958
```

If we had naively noisy-OR'd all three as if independent:

```text
1 − (1 − 0.72)(1 − 0.63)(1 − 0.85) = 0.984
```

The naive number is higher **because `S`'s evidence got counted twice** through
both A and B. The 0.958 is the honest answer; the 0.984 is overconfidence
manufactured by ignoring the shared ancestor. That gap is the entire point of
the calculus.

## Why product + noisy-OR (and not the old mix)

Product (AND) and noisy-OR (OR) are **De Morgan duals** in the same probabilistic
"noisy logic": noisy-OR is literally `1 − product-of-complements`. They belong to
one algebra, so chaining and corroborating compose coherently no matter how the
graph is shaped.

The earlier `min + noisy-OR` mix was replaced precisely because it was *not* one
algebra. `min` is a Zadeh/fuzzy t-norm; noisy-OR is a probabilistic co-norm. Mixing
them meant the AND-step and the OR-step answered to different definitions of
"combine," and the numbers stopped meaning one consistent thing. Product is the AND
that is dual to the noisy-OR we already use — so the two halves finally speak the
same language.

## Open problem — detecting independence at scale

Steps 1, 3, and 4 are cheap. **Step 2 — partitioning paths into
independence groups — is the hard, unsolved part at scale.** Deciding whether
two paths share a load-bearing ancestor is itself a graph problem, and doing it
on every query over a large graph is expensive.

The principled solution space is **region-based / loop-corrected belief
propagation** — generalized BP and its relatives [Yedidia, Freeman & Weiss 2005;
Mooij & Kappen 2007] — which correct overcounting explicitly on regions rather
than hoping paths are independent. Whether a cheap-enough approximation exists
for this project's online traversal is one of the named open problems; it is the
unsolved edge of ADR-3, not a settled detail.

**Measured (T1 spike, 2026-06).** The saturation spike
([swarm ADR-3](../../swarm/docs/decisions/0003-confidence-traversal-bounding.md);
bench numbers in `swarm/docs/design/confidence-saturation-spike.md`) found that
**grouping is not the wall — path *enumeration* is.** The recursive-CTE traversal
materializes one row per path (`fanout^depth`) and collapses by ~72k edges at
fanout 8 / depth 9, long before grouping runs; grouping itself is ~`O(P)` given
the path set. So the kernel fix is **node-bounded traversal** (best-confidence-
per-node relaxation, identical result for single-source), and region-based BP for
large-scale multi-origin partitioning is **deferred, not on the critical path** —
it is the cheap step, optimized only once multi-origin corroboration at scale
exists.

## Independence in the strength dimension

The independence trap is not confined to confidence aggregation. It reappears,
unsolved, in the **strength** dimension — edge reinforcement (ADR-9).

An edge's `seen_count` increments once per distinct `(edge, provenance)` event
(the `edge_provenance` unique guard in `Swarm.Graph.Store`). That guard is exact
for one hazard — the *same* event must not count twice — and it closes the
endogenous confirmation loop, where the system re-derives an edge and counts its
own output. It says nothing about a different hazard: **distinct events that are
not independent evidence.** Fifty documents that each restate one rumour carry
fifty distinct provenance keys and reinforce the edge fifty times, exactly as if
fifty independent witnesses had confirmed it.

This is the shared-ancestor overcounting of the section above, moved from "paths
sharing a source" to "reinforcements sharing a source." The confidence side
corrects it (max within a shared-ancestor group). The strength side, today, does
**not**: it has no per-source ceiling and no lineage grouping.

What holds it in check now is mitigation, not correction:

- **Hill saturation** `f(n) = log(1+n)/(log(1+n)+S)` bounds the inflation — a
  burst of correlated reinforcements saturates rather than scaling linearly;
- **decay is the dominant pole** — a one-off correlated burst is stirred in and
  then forgotten.

So a *transient* correlated burst is largely absorbed. The residual hazard is a
*sustained* one: a connector that re-emits derivatives of a single source with a
fresh provenance key each time keeps refreshing `last_seen` and pins the edge
near saturation indefinitely — converting a decay-dominated edge into an
effectively immortal one, with no genuine new evidence behind it. Nothing in
`ports.md` or ingestion currently requires a provenance key to track *evidential
origin* rather than *emission instance*.

### Solution space (no decision yet)

- **Connector provenance contract (cheapest).** Require provenance keys to be
  derived from evidential origin (content/source identity), not emission
  time/instance: re-emitting the same fact must reuse the same key. A rule in
  `ports.md`, enforced at ingest, pushing the problem to the boundary where
  origin is actually known.
- **Per-source reinforcement cap.** Cap an edge's reinforcement contribution per
  source/lineage — the direct mirror of ADR-3's "max within a shared-ancestor
  group," applied to strength. One source cannot reinforce past a ceiling no
  matter how many derivative events it emits.
- **Lineage-aware reinforcement.** Cluster provenance by source lineage before
  counting — the strength-dimension twin of partitioning paths into independence
  groups, and just as hard at scale (the same open problem).

The first is a boundary contract; the second is the elegant in-kernel correction
(same algebra as ADR-3); the third is the principled-but-expensive general
solution. Governed by
[ADR-9](../decisions/0009-stigmergic-loop-stability.md) — the strength-side
instance of the independence open problem named above.

## What this is not

- **Not fuzzy min/max logic.** The within-group max is a deliberate
  independence-correction, not a general combinator; the general combinators are
  product and noisy-OR.
- **Not a vote count.** Corroboration is noisy-OR over *independent* evidence,
  not a tally; ten paths from one source are still one source.
- **Independence is load-bearing, not optional.** Skipping step 2 doesn't make the
  calculus simpler — it makes it wrong, in the specific direction of false
  confidence.

## Related

- [ADR-3: Confidence Calculus](../decisions/0003-confidence-calculus.md)
  records this as a locked decision.
- [../reference/glossary.md](../reference/glossary.md) — §2 defines the terms used
  here (noisy-OR, log-space product, overcounting, path independence) and their
  sources.
- [../reference/bibliography.md](../reference/bibliography.md) — §6 lists the loopy-BP
  and overcounting literature this draws on.
