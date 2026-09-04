# ADR-22: The environment observation port

Workspace-level. Builds on ADR-21 (connector packaging and scheduling — kernel drives the
loop, adapter is a sidecar, finite scheduled run, skip ledger and `truncated` preserved),
ADR-13 (evidence kinds and supersession) and ADR-20 (project access and scopes).

**A note on the word "port", because the first draft of this ADR got it wrong.**
`architecture/ports.md` keeps port kinds deliberately generic — `connector`, `tool`,
`worker`, `channel`, `model`, `skill` — and puts specific systems in the *domain* half of
a plugin's name. An environment observer ingests from an external system, so it is a
`connector`; inventing an `environment_observation` **kind** would push a specific system
into the kernel's vocabulary, which that document forbids. What is genuinely new is a
narrower **behavioural contract shared by several connectors**, and that is what this ADR
defines. The registry gains a profile, not a kind.

## Status

**Proposed.** Design only — no access requested, no credentials, nothing run against
production. One decorrelated critic consulted **before** this was written
(`qwen3:14b`, local, a different model family); its two FLAWED findings are folded in
below and named as such. Two of the four questions put to it — (c) whether one port with
three adapters is right, and (d) what breaks only after months — **came back unanswered**,
so those sections are my reasoning alone and are the weakest part of this document.

## Context

The estate's written records do not say which machine runs a named service in any form a
machine can check. Measured twice, independently, on the same 19-question frozen set:

| source of the service→host link | result |
| --- | --- |
| corpus title-equality (`board/journal.md`, milestone 6) | 2 / 19, and both still need a query→document hop |
| IaC service groups, API-graded (milestone 8, re-verified) | **1 / 19** |

The re-verification matters, because the first reading of the IaC number was wrong in a
way worth remembering: `5/19` counted how often the rule *applied*, not whether it
produced a correct link. Graded against the hypervisor API it is 1/19. Checked again for
this ADR against the specific rows: `dependency_track` links and is correct;
`ssp_vms` resolves to exactly the right host but the question says
"Self-Service-Password", so reaching it needs an acronym expansion — a similarity
judgement; and `agencies_wireguard` would have linked **wrongly**, to agency endpoints at
two other sites, had the rule been laxer.

So the link is not there to be found. **An observation channel does not find it — it makes
it.** A machine that reports "a unit named `keycloak.service` is active here" states a fact
nobody wrote down.

## Decision

Define **one shared contract**, the *environment-observation profile*, implemented by
several `connector` plugins. A conforming connector answers one question — *what is
actually running in this environment* — and reports observations, never claims. ADR-21's contract is unchanged: kernel drives, adapter is a sidecar, runs
are finite and scheduled, skips are recorded, `truncated` is preserved.

Three implementations, named per the `<domain>_<kind>` rule — `localenv_connector`,
`k8s_connector`, `sshhost_connector` — in this build order, and the first is load-bearing:

1. **Local / self.** The environment Swarm itself runs in. No access grant, no
   negotiation, and it exercises the whole port. It also answers something we cannot
   answer today: what Swarm's own deployment consists of.
2. **Kubernetes.** The API reports workloads and placement directly. No shell.
3. **Remote host over SSH.** Last, because it is the only one with a shell on production.

### D1 — Identity. The critic was right, and the design changed

*Proposed:* key an environment by an identifier it reports about itself and that survives
renaming — `/etc/machine-id`, the `kube-system` namespace UID, a container UID — as
`env:<kind>:<uuid>`, with hostnames and site labels as attributes, never identity.

*The critic returned FLAWED, and correctly:* that solves identity **within** the port and
does nothing to tie `env:host:<uuid>` to the VM the hypervisor reports. The hypervisor
knows a guest by its own vmid and knows nothing about a machine-id. Asserting the tie on a
matching hostname would be the ADR-17 error again, one layer along.

*Resolution, and it is a restructuring rather than a patch:*

- The port does **not** claim to solve the environment↔inventory tie. `env:` subjects are
  their own keyspace. An observation says *this environment has this unit active*, and
  says nothing about which Proxmox guest the environment is.
- **Adapter 1 does not need the tie at all** — Swarm's own deployment is the subject, and
  the question "what does our deployment consist of" is answered entirely inside the
  `env:` keyspace. This is the second reason to build it first.
- For adapters 2 and 3 the tie is an **open precondition with a named check**: does the
  Proxmox guest-agent endpoint report an identifier the guest also reports about itself?
  At least one site's API token already carries the guest-agent audit privilege, so the
  check is cheap — but it is a read against production and this campaign does not make
  one. **Until that check returns a shared identifier, adapters 2 and 3 stay unbuilt.** No lexical fallback is permitted; if
  the check fails, the tie needs a declaration from a source that states it, and that is a
  different decision.

### D2 — Observation versus declaration

Adapter output is `evidence_kind: observation`, `valid_time` = the observation instant,
with a per-run completeness flag. Absence closes a fact **only** when the run completed; a
partial run must never become "the unit is gone". Where an observation and an IaC
declaration disagree, the observation takes precedence for what-is-running and the
declaration is recorded as **stale declaration**, never averaged into a corroboration
conflict (`board/todo/source-authority.md`). The environment has **no** authority over
purpose or intent, ever — those stay testimony.

*Critic suggested* adding a confidence score to express partial-run uncertainty.
**Rejected, deliberately.** A scalar is exactly how a wrong observation gets averaged into
a right one; this project spent a day building a metric that refuses to do that.
Completeness stays a hard flag: complete, or absence means nothing.

### D3 — What is observed. The critic's second FLAWED, resolved more narrowly than proposed

*Proposed:* v1 observes which service units are active.

*The critic's failure case is real:* a reverse proxy fronting a service that runs
elsewhere is active locally, and "X runs Keycloak" would be false. Its suggested fix was
to widen scope — dependencies, endpoints, health. **Rejected**: that is how the filing
cabinet got built, and it does not remove the inference, it buries it.

*Resolution — report the unit, not the service:*

- the observation is literally `env:<id> has_active_unit "keycloak.service"`, with the
  command that produced it. That is directly observed and cannot be false unless the
  machine lied.
- **no adapter ever emits "runs service Keycloak".** Going from a unit name to a service
  identity is an inference, it belongs to a later and separately evidenced step, and it
  must be refused where ambiguous — the same rule the document→host linker already
  follows.
- a listening socket is a *different* observation from a unit and is not evidence for the
  same claim; v1 collects units only.

This keeps v1 minimal and keeps the proxy case from becoming a false fact rather than
merely an unresolved one.

### D4 — Client machines

No central registry: an environment self-presents its identity, and the kernel accepts or
rejects it. A client's observations land in a scope only that client can read, which
ADR-20's per-source scopes already express. If the port only made sense for infrastructure
we own, it would be the wrong port — a laptop is an environment like any other.

Critic: SOUND, no change.

### D5 — The cost of a wrong observation

An adapter reports only what it directly observed, never inferred; every observation
records the exact command or API call that produced it; an adapter may assert facts only
about **its own** environment. This is the same provenance discipline the kernel now emits
for answers (`Swarm.Core` per-answer provenance record) — an observation whose origin
cannot be named is not an observation.

Critic: SOUND, no change.

## The two questions no critic answered

Stated as mine, so they can be attacked.

**(c) One port or two?** A shell prober and an API prober differ in failure modes, not in
what they assert. Both answer "what is active here" and both can be partial. I expect the
abstraction to leak at **completeness**: a Kubernetes list call is atomic and either
returns the full set or fails, while a shell run is a sequence that can half-succeed, so
"complete" means different things and a single boolean will be doing more work on one side
than the other. The port survives that only if completeness is per-*observation-class*
rather than per-run. I have not designed that, and it is the first thing to test with
adapter 1.

**(d) What breaks after months?** My guess: **unit names drift and nothing notices.** A
unit renamed during a migration produces a new `has_active_unit` fact and silently closes
the old one; with the environment authoritative, the graph will faithfully record that the
old thing stopped and a new thing started, and no one will be told they are the same
service. That is the ADR-17 identity problem arriving from the other direction, and the
port as designed has no answer to it.

## Consequences

- A fourth source, added after two measurements showed the third could not close the gap
  — the campaign's own rule (do not widen coverage before checking coherence) is satisfied
  by that evidence, not waived.
- `env:` is a new keyspace deliberately not joined to `net:host:`. Until D1's precondition
  is checked, the port improves what we know about *our own* deployment and nothing about
  the estate. That is a smaller claim than "we solved the join", and it is the true one.
- Nothing here touches the serve path. A new domain reading `env:` facts is a later
  decision with its own precedence question (`domain.ex` first-match, ADR-17 rejection).

## The local adapter, concretely

Enough to build; nothing here requires an access grant.

| | |
| --- | --- |
| identity | `/etc/machine-id`, or the container's own UID when Swarm runs containerised; refuse to run if neither is readable rather than inventing one |
| observed | active systemd units by exact name and state, from one `systemctl list-units --type=service --state=active --output=json` |
| emitted | `env:<kind>:<uuid> has_active_unit <unit-name>`, `evidence_kind: observation`, `valid_time` = run instant, provenance = the exact argv |
| completeness | the run is complete iff the single command exited 0 and its output parsed; anything else marks the run partial and **closes nothing** |
| skips | a unit whose name does not parse is a recorded skip, never a silent drop (ADR-21) |
| scope | the source scope of the `environment` source, per ADR-20 |
| never | no shell pipeline, no config reads, no process table, no network probing, no writes |

Test before trusting: the adapter must be exercised against a fixture environment with a
known unit set, including a partial-run case that must close nothing — the same shape as
the learner-eval fixtures, and for the same reason.

## What must be true before this is Accepted

1. A decorrelated critic on **(c)** and **(d)**, which this round did not cover. Note
   that (c) — one contract or two — is now partly a question about the *profile*, not a
   port kind, which lowers its stakes: two profiles inside one kind cost far less than two
   kinds.
2. Agreement that `env:` staying unjoined from `net:host:` is acceptable for v1 — it is
   the honest position, but it means the port does not close the measured gap yet.
3. The D1 precondition check named and scheduled, as a read the operator authorises, not
   as an assumption this ADR is allowed to make.
