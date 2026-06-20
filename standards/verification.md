# Verification

How output gets checked before it's trusted. This is normative: the levels and the
trust order below are not suggestions.

The governing rule is the same one the swarm itself runs on — **reward comes from
external ground truth, never from the system grading itself.** A model reviewing its
own work is the weakest signal there is. Everything here is built to push judgment
*outward*, toward signals a model can't fake.

## Three levels

Apply them in order. Each later level is more expensive and rarer than the last.

### 1. Criteria up front

Before building, the criteria for "done" are written down — see the `/criteria`
command. This is the cheapest and highest-leverage level: without an explicit target,
the next two levels have nothing to check against, and review collapses into "looks
fine to me." No criteria, no build.

### 2. Independent critic

A second model reviews the output against the criteria. The critic **must be a
different model from the author** — a model reviewing its own output is blind to its
own failure patterns, so self-review is barely above no review at all. The point is a
second *mind*, not a second *pass*.

### 3. External signal

Wherever the criteria can be expressed mechanically — a test passes, it compiles, a
benchmark holds, a `task` target goes green — that signal outranks every opinion,
including a unanimous panel of models. This is ground truth. Reach for it first and
express as many criteria as possible in its terms; a criterion you can turn into a
failing test is worth more than three you can only argue about.

## The model roster

We have three independent sources of judgment available. They are deliberately
different families, because diversity of error is the whole value:

- **Claude (Opus)** — author and planner. Writes the spec, writes the code.
- **codex (local)** — independent code critic. Different model family → different
  blind spots. Runs locally, so it's cheap enough to call on every meaningful diff.
- **gemini + local models (Spark)** — the third voice, for architectural calls and
  anywhere a third independent read adds signal. Local models are cheap enough for
  frequent low-stakes sanity checks; gemini is the heavier escalation.

**The author never grades itself.** If Claude wrote it, Claude is not the verifier.

## Disagreement is signal, not noise

This is the **consilium** pattern (ADR) applied to our own dev loop. When codex and
Claude reach the same conclusion, confidence is high. When they disagree, that is not
a tie to be broken by picking one — it is a flag that says *stop and look here*.
Disagreement marks exactly the spots where a human should spend attention. Keep the
disagreement as a confidence signal; do not average it away or silently defer to one
model.

## Trust hierarchy

When sources conflict, trust in this strict order:

1. **External signal** (tests, compile, benchmark, green `task`) — ground truth.
2. **Independent models disagreeing** — not an answer, a *flag*: escalate to a human.
3. **A single independent critic** — useful judgment, but one opinion.
4. **Author self-review** — weakest. Acceptable only for trivial, reversible work.

A higher level always overrides a lower one. A passing test beats a model's
confident objection; two models agreeing does not beat a failing test.

## Cost asymmetry

Same principle as the swarm: cheap checks run constantly, expensive checks are a rare,
deliberate escalation. Local models and codex are cheap — run them often, on every
diff worth reviewing. gemini and full panel reviews are the escalation, reserved for
architectural decisions and the must-pass gates. Don't spend a consilium on a typo;
don't ship an architecture change on a single self-review.

## What this is not

Not a substitute for the criteria — review without a target is theater. Not a vote to
be tallied — external signal isn't outranked by a model majority. Not self-grading —
if there's no source more external than the author in the loop, the work is unverified,
regardless of how confident the author sounds.
