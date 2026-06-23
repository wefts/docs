# Presentation Determinism

How output is rendered, across every channel. Normative.

## The rule

**The kernel emits structured facts; channels render them deterministically.** A
model never chooses the presentation of a value, link, id, count, or list.

- The kernel's `Core.ask` returns a typed result — `answer`, `status`
  (`found / not_found / partial / error`, see
  [ADR-6 (swarm)](../../swarm/docs/decisions/0006-answer-result-algebra.md)),
  `confidence`, and `citations` (source / ref / confidence).
- A **channel** (CLI, web, chat) formats those fields with deterministic code:
  tables, labels, styles, ordering. It does not ask a model how to format them,
  and it **never parses the answer prose to recover a value that is already a
  structured field**.
- Anything a user acts on — a link, an id, a ticket number, a file path, a
  count — is rendered **verbatim from its structured field**, never re-spelled by
  a model.
- A channel must **escape verbatim values for its medium** (Rich markup, HTML,
  Markdown) so a value containing `[`, `<`, `&`, or a backtick stays the value
  and is not interpreted as formatting — verbatim must survive rendering.

### The `answer` string is the single voice — a carve-out

`Core` is the **single voice** (Domain 11): it composes the one human-facing
`answer` string. For an escalation that string is the model-synthesized verdict;
for not-found / partial / a tool hit it is a deterministic *kernel* template
("I found nothing for X", "Found N item(s)"). That kernel-authored prose is a
**deliberate exception** to "only channels render", justified by the single-voice
rule — but it is the kernel's **default** rendering, not the source of truth.

The source of truth is the **structured fields** (`status`, `citations`,
`confidence`, and the query the channel itself holds). A channel renders the
default `answer` as-is, **or** — to change register, verbosity, or **language**
(T9 persona) — composes its own prose **from the structured fields**, never by
parsing or translating the kernel prose. The default voice is English; a
localized channel re-phrases from the fields. This is what stops the language
drift: the facts are language-neutral fields, and the kernel prose is only a
fallback skin over them.

## Why (determinism > prompt)

The prototype relearned this five times: backticked links that broke, stray
emoji, a brevity re-rank that silently hurt recall, a UK→RU language drift, and
"open the page" prose where the user needed the actual value. Every fix moved the
decision out of the model and into deterministic channel code. A prompt is a
request; code is a guarantee. Presentation is a guarantee.

## Persona is not facts

A channel (or a `skill`) may choose **register/persona** — tone, verbosity,
language, DM-vs-public sharpness (see T9). That is allowed and context-dependent.
But persona styles the *prose around* the facts; it never decides, reorders, or
reformats the facts themselves. The status, the citations, the values are the
kernel's; the voice is the channel's.

## Boundary (what lives where)

| Concern | Owner |
| --- | --- |
| The facts (source of truth): `status`, `citations`, `confidence` | kernel (`Core`) |
| The single-voice `answer` string (default prose) | kernel (`Core`) — overridable by a channel re-phrasing **from the fields** |
| Rendering: tables, labels, styles, ordering, status banners, escaping | channel |
| Register / persona / language (re-phrased from the fields) | channel / `skill` (T9) |
| Choosing or reformatting a value/link/id | **nobody — it is rendered verbatim** |

No presentation logic leaks into the kernel; no fact-selection leaks into the
channel.

## Reference channel

The CLI (`swarm/cli`) is the reference: it renders `AskResponse` from structured
fields only — the status maps to a fixed label+style (`_render_status`), citations
render in a table from `source/ref/confidence`, confidence drives a semantic
color. No emoji, left-aligned, deterministic. Tests assert a value is rendered
verbatim from its field, and the status from the structured field — not inferred
from prose. Future channels inherit this standard.
