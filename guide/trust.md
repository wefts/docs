# How sure is Swarm?

> **Plain-language guide.** The precise rules live in the
> [confidence calculus](../architecture/confidence-calculus.md) and
> [claim typing](../../swarm/docs/decisions/0011-graph-zones-and-claim-typing.md).

Not everything Swarm knows is equally certain. Every card carries a **trust score**, and
Swarm raises or lowers it as evidence comes in. This page explains how — and the rules
that stop it from fooling itself.

## Trust grows from *independent* agreement

If several **separate** sources say the same thing, confidence should go up — that is real
corroboration. The key word is *separate*. Two echoes of one source are not two witnesses;
they are one witness, twice. (What counts as *one* source is its own question:
[origins.md](origins.md).)

This is why the design insists on **one source, one card** (see
[memory-model.md](memory-model.md)). If a single page were chopped into fifty cards, those
fifty would later look like fifty independent witnesses to the same fact, and Swarm would
talk itself into false certainty. Keeping the card coarse keeps the count honest.

```mermaid
flowchart TD
    F["one fact"]
    F --> A["source A"]
    F --> B["source B"]
    F --> C["source C"]
    A --> U["3 independent witnesses → confidence up"]
    B --> U
    C --> U
```

## Two rules that prevent fake confidence

- **Chunks do not vote.** A passage of text can be *found* by a search, but it is not a
  witness on its own. Only the card it belongs to carries weight. (This is the one idea
  the whole guide hangs on.)
- **Guesses do not count as independent witnesses.** When a language model later infers
  something from existing cards, that inference is marked as a *claim*, not fresh
  evidence. A burst of model-generated claims about one fact collapses to a single voice,
  so the model cannot agree with itself into certainty. (That inference step is the
  cognitive layer: [cognition.md](cognition.md).)

## Some sources are trusted more than others

Where a card came from also feeds its trust. An official reference and a fan wiki can be
weighted differently — the origin (see [provenance.md](provenance.md)) is part of the
score, not just a label.

## What's built, and what's still hard

The blunt version of this problem is **closed**: Swarm counts distinct **origins**, not
documents, so fifty copies of one rumour reinforce nothing ([origins.md](origins.md)), and
model-made claims collapse to one voice. Those defenses are built and working, not aspirational.

What stays genuinely hard is the subtle case — sources quietly *derived from one another*
(A copies B copies C), which look independent until you trace the lineage. Doing that across
a large corpus is expensive, so it is **deferred** until real data actually overlaps that
way. So when this guide says Swarm is "sure", still read it as "sure given what it can
currently tell apart" — but the everyday trap is now shut, not open.

Next: [cognition.md](cognition.md).
