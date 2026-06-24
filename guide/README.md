# Swarm, in plain words

This folder explains how Swarm works in **plain English**, for readers from any
background — not just the people who build the kernel. It trades precision for clarity on
purpose.

> **This is the friendly layer.** It explains *ideas*, not exact contracts. The
> authoritative, precise versions live in the specialist canon:
> [`docs/architecture/`](../architecture/) (system shape) and
> [`swarm/docs/`](../../swarm/docs/) (decisions and specs). Each page here links down to
> its precise source. Where the two ever disagree, the canon wins and this page is stale.
> For the exact definition of any term, see the [glossary](../reference/glossary.md).

## Who this is for

Anyone who wants to understand *what Swarm does and why* without reading the source or the
decision records. We assume you know the basics — a database, an index, a graph — but not
Swarm's own vocabulary. Every term is explained before it is used.

## What Swarm is, in one paragraph

Swarm is a local-first memory. Its goal is **search by meaning, not by letters**: to know
that two pages about one thing are a single memory, and that two spellings of a name are
one name. It stores knowledge as a web of small **cards** (a graph), with the actual text
kept beside them and indexed two ways — by word and by meaning.

## The pages

Read them in order, or jump to what you need:

1. [memory-model.md](memory-model.md) — the two things Swarm stores: a *card* and its
   *text*.
2. [search.md](search.md) — the two indexes that turn a query into matches: by word and by
   meaning.
3. [the-graph.md](the-graph.md) — the web of links, and why finding things is not the same
   as walking it.
4. [trust.md](trust.md) — how sure Swarm is, and why "one source, one card" keeps it
   honest.
5. [provenance.md](provenance.md) — where a memory came from, and how to ask within one
   source.

## The one idea to keep

If you remember nothing else: **a card is a memory and can carry weight; a chunk of text
can be *found* but cannot *vote*.** Most of the design follows from that.
