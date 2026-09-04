# Verification

How output gets checked before it's trusted. This is normative: the levels and
the trust order below are not suggestions.

The governing rule is the same one the swarm itself runs on — **reward comes
from external ground truth, never from the system grading itself.** A model
reviewing its own work is the weakest signal there is. Everything here is built
to push judgment *outward*, toward signals a model can't fake.

## Three levels

Apply them in order. Each later level is more expensive and rarer than the last.

### 1. Criteria up front

Before building, the criteria for "done" are written down — see the
`/criteria` command. This is the cheapest and highest-leverage level: without an
explicit target, the next two levels have nothing to check against, and review
collapses into "looks fine to me." No criteria, no build.

### 2. Independent critic

A second model reviews the output against the criteria. The critic **must be a
different model from the author** — a model reviewing its own output is blind to
its own failure patterns, so self-review is barely above no review at all. The
point is a second *mind*, not a second *pass*.

### 3. External signal

Wherever the criteria can be expressed mechanically — a test passes, it
compiles, a benchmark holds, a `task` target goes green — that signal outranks
every opinion, including a unanimous panel of models. This is ground truth.
Reach for it first and express as many criteria as possible in its terms; a
criterion you can turn into a failing test is worth more than three you can only
argue about.

## The model roster

We have three independent sources of judgment available. They are deliberately
different families, because diversity of error is the whole value:

- **Claude (Opus)** — author and planner. Writes the spec, writes the code.
- **codex (local)** — independent code critic. Different model family →
  different blind spots. Runs locally, so it's cheap enough to call on every
  meaningful diff.
- **gemini + local models (Spark)** — the third voice, for architectural calls
  and anywhere a third independent read adds signal. Local models are cheap
  enough for frequent low-stakes sanity checks; gemini is the heavier
  escalation.

**The author never grades itself.** If Claude wrote it, Claude is not the verifier.

## Disagreement is signal, not noise

This is the **consilium** pattern (ADR) applied to our own dev loop. When codex
and Claude reach the same conclusion, confidence is high. When they disagree,
that is not a tie to be broken by picking one — it is a flag that says *stop and
look here*. Disagreement marks exactly the spots where a human should spend
attention. Keep the disagreement as a confidence signal; do not average it away
or silently defer to one model.

## Trust hierarchy

When sources conflict, trust in this strict order:

1. **External signal** (tests, compile, benchmark, green `task`) — ground truth.
2. **Independent models disagreeing** — not an answer, a *flag*: escalate to a
   human.
3. **A single independent critic** — useful judgment, but one opinion.
4. **Author self-review** — weakest. Acceptable only for trivial, reversible work.

A higher level always overrides a lower one. A passing test beats a model's
confident objection; two models agreeing does not beat a failing test.

## A check that did not run is not a check that passed

Verification has a failure mode worse than a wrong answer: a check that **silently
did not happen**, whose absence then reads as a pass. Two rules follow, both learned
the expensive way.

### Read the artifact, not the clock

When auditing whether a measurement is trustworthy, **open its output**. Do not decide
from timestamps against the window an outage is believed to span.

The event that taught this: a model daemon was SIGKILLed by `earlyoom`, which kills the
largest process before the kernel OOM killer and therefore never sets Docker's
`OOMKilled` flag. Docker recorded `FinishedAt` as the moment the container finished
exiting — but two calls that **precede** that timestamp had already failed with
`RemoteDisconnected`, because they were the requests in flight while the process was
being torn down. The outage began at least fifty seconds before the timestamp that
appeared to define it. **An audit keyed to `FinishedAt` cleared two artifacts that had
already failed.** Only reading the files caught it.

Generalise past that one cause: a timestamp bounds when a process *finished*, never when
it *stopped working*. Degradation precedes death, and the artifact is the only witness to
it. So an audit is not done until each result has been checked for a signature that only
a working dependency could have produced — an error status, a differentiated output, a
value that varies with the condition it was supposed to vary with.

### A guard must assert the dependency, not its side effects

Health checks must test the thing they claim to protect. The guard around the same
measurements printed `MemAvailable` and memory PSI, and both looked *excellent* right
after the kill — precisely **because** the largest process had just been killed. The
guard was reading the consequence of the failure as evidence of health.

So: assert liveness of the dependency itself, require it before **and after** the work,
and **abort rather than warn** — a run against a dead dependency must not be able to
write a number. Where the work is already on disk before the after-check can run, mark
it void beside itself rather than leaving a half-real file to be found later and trusted.

**And confirm the check ran against the thing you meant.** `leak_scan.sh` derives its
repo root from its own path (`cd $(dirname $0)/..`), so invoking one repo's copy from
inside another silently scans *the script's* repo and reports a confident OK about a repo
it never opened. Four such OKs were recorded in one day for `swarm/` and `docs/` that had
all scanned `hive/`, and `docs/` — a public repo — turned out to have never been scanned at
all. A green check names its subject, or it is not evidence about your subject.

And when an unattended call is the check, **inspect its exit status**, never the
existence of its output file. Three critic reviews in that episode were a traceback, a
traceback, and zero bytes, sitting in a directory looking like a council.

## Cost asymmetry

Same principle as the swarm: cheap checks run constantly, expensive checks are a
rare, deliberate escalation. Local models and codex are cheap — run them often,
on every diff worth reviewing. gemini and full panel reviews are the escalation,
reserved for architectural decisions and the must-pass gates. Don't spend a
consilium on a typo; don't ship an architecture change on a single self-review.

## What this is not

Not a substitute for the criteria — review without a target is theater. Not a
vote to be tallied — external signal isn't outranked by a model majority. Not
self-grading — if there's no source more external than the author in the loop,
the work is unverified, regardless of how confident the author sounds.
