# Agent governance mechanisms

A companion catalog to [mechanisms.md](mechanisms.md), for the containment layer:
how an agent or plugin is bounded, admitted, budgeted, and audited so it cannot
exceed what it was declared to do. Each entry is a generic mechanism, then how it
maps onto our kernel and which port or decision it touches, then the caveat. As in
the companion, these are candidates to adopt and reconcile against the kernel, not
settled canon.

The through-line: **a capability is declared once at the boundary, validated before
it runs, and bounded by a wall the agent's own code cannot reason around.** The
agent is never trusted to limit itself — the limit lives outside it.

## The premise: containment by design, not by wrapper

A general lesson worth stating before the mechanisms. Agent frameworks supply
*behaviour* — a loop of model calls, retrieval, and tool use — but not *control*:
no isolation, no enforced budgets, no audit, no egress bounds. The common industry
answer is to wrap an otherwise-unconstrained agent in an external platform that
imposes those after the fact. Our answer is the inverse: the constraints are
structural — the kernel never imports plugin source, side effects are explicit and
auditable, adapter failure cannot bring down the kernel, and outward actions pass a
single Tool gateway. The mechanisms below make that posture enforceable rather than
merely intended.

A note on the coordination boundary, since it is where containment is easiest to
lose: direct agent-to-agent messaging is **not** part of this model (ADR-2). Agents
coordinate through traces in the shared graph, not by calling each other. That
choice removes a whole class of uncontrolled propagation up front, so the governance
here bounds each agent against the *substrate and the outside world*, not against an
unbounded mesh of peers.

## Declared capability contract

A capability is defined exactly once, at the boundary where the system's identity is
known, so no agent re-invents how to use it. The contract pins *how* the capability
is called — methods, parameters, side-effect class — and every caller invokes the
declared method rather than rediscovering the protocol per agent. A wrapper over an
external API (for example an MCP server fronting a ticketing or asset system) is this
pattern: the integration's methods are declared once and correctly, and become the
only sanctioned way in.

- *Maps to:* the `connector` and `tool` ports and the manifest's `declared
  capabilities` / `safety class for side effects`. The manifest is the declaration;
  the port is the contract it implements.
- *Caveat:* the contract fixes *how* a capability is called, not *who* may call it or
  *within what limits* — that is the admission and budget mechanisms below. A declared
  capability with no admission gate is a well-documented open door.

## Validate-then-admit

The declaration is not just documentation read by a human — it is run through a
policy engine *before* the agent is allowed to execute. The spec is validated
(required fields present, declared egress and tools within allowed sets, no
forbidden destinations), and then admitted, mutated, or rejected. A policy change
can re-admit or block an already-declared agent at the boundary without editing its
internals.

- *Maps to:* the manifest minimum in `ports.md`; lands hardest at the WASM isolation
  tier, where untrusted skills must clear an enforced precondition, not a convention.
  It is the same policy-vs-mechanism split as `guardrails.md` (🔒 enforced vs 📝
  written), applied at load time.
- *Caveat:* admission validates the *declaration*, not the runtime behaviour — it
  proves the agent asked for nothing forbidden, not that it does nothing forbidden.
  Pair it with the egress and audit mechanisms, which catch behaviour at run time.

## Budget as an external wall

Spend limits (tokens, requests, cost) are enforced outside the agent, by a component
the agent's code cannot bypass. The pattern: a proxy mints a per-agent credential
that is *already* capped — a virtual key bounded on tokens and request rate — so the
agent calls the model through a key that simply cannot exceed its allowance. The
limit is a wall, not a number the agent is trusted to respect; a runaway loop hits
the cap instead of the bill.

- *Maps to:* the budget decision (ADR-7 — per-escalation ceiling, the runaway path
  refuses) and the Model port; reinforces the gate/consilium cost asymmetry, where
  expensive escalation must stay bounded regardless of how much cheap work fans in.
- *Caveat:* a hard cap bounds spend but not wall-clock — an agent can still stall
  inside its allowance (slow upstream, no progress). Time-to-execution needs its own
  bound; the budget does not supply it.

## Bounded egress

What an agent can reach is declared and enforced at the network boundary, default-
deny: it may talk only to the destinations its spec allows, and nothing else — not an
arbitrary endpoint, not a tool it found at runtime, not an exfiltration target. The
allowed set is part of the declaration and enforced by the platform, not left to the
agent to honour.

- *Maps to:* the default-deny visibility model (ADR-5), now extended outward — ADR-5
  bounds what an agent may *see* in the graph; egress policy bounds where it may
  *reach*. Same fail-safe shape: an empty allowed set reaches nothing.
- *Caveat:* egress rules bound destinations, not payloads — a permitted destination
  can still receive data it shouldn't. Couple egress with the visibility scope that
  governs what data the agent holds in the first place.

## Immutable, reasoning-level audit

Every effectful step — and the reasoning that led to it — is written to a store the
agent cannot alter or erase. The point is the immutability: an agent that "decides to
clean up after itself" must not be able to. Trace inputs, prompts, model and
parameters, tool calls, and the reasoning trace, into append-only storage.

- *Maps to:* the provenance/origin contract (ADR-13) and default-deny visibility
  (ADR-5); on the substrate this is the same append-only, never-mutate discipline as
  event sourcing (glossary §1), which the kernel already relies on as the
  reconsolidation antidote.
- *Caveat:* audit is only as trustworthy as its write path is tamper-proof. If the
  agent has write access to the audit sink, the audit is theatre — the sink must be
  outside the agent's reach by construction, like `hive/data/` is off-limits to direct
  edits.

## How the mechanisms compose

These are halves of pairs, and the value is in the composition. The **declared
capability contract** fixes *how* a capability is called; **validate-then-admit**
fixes *who* may call it; the **budget wall** and **bounded egress** fix *within what
limits*; **immutable audit** fixes *what is provable after the fact*. A correctly
built capability wrapper with no admission gate is an open door; an admission gate
over an undeclared capability has nothing to check; a budget without audit cannot be
reconciled. Adopt them as a set, not à la carte.

## Compliance vocabulary (when local-first is left behind)

The external regulatory frames (EU AI Act, GDPR, NIST AI RMF, OWASP) converge on a
small set of demands this catalog already answers: immutable audit of reasoning,
data-protection scoping, declared and bounded capability. They are not in scope while
the system is local-first, but they are a ready checklist for the visibility-filter
threat model (OP#4) if the system ever serves data across a trust boundary.

## Reconcile against the kernel

Parts of this already exist — default-deny visibility, the provenance contract, the
Tool gateway, the 🔒/📝 guardrail split, the ADR-7 ceiling. The governance-specific
gaps (validate-then-admit at load time, an external budget wall, declared egress
enforcement) are the candidates here. Before adopting an entry, diff it against the
running code; this catalog is the *why*, the kernel is the *what is true now*.