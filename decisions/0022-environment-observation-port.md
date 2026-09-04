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
production.

Two critic rounds, both **before** Accepted. **Neither was decorrelated, and that is a
known gap in this ADR's review, not a claim it meets the bar** — see *Review breadth* below.

- *round 1* (`qwen3:14b`, local, different family), consulted before this was first
  written: FLAWED on identity and on what-is-observed. Folded in.
- *round 2*, on the written ADR: **FLAWED**, with one survival — **the connector-kind
  decision is sound, so "a profile, not a new port kind" stands.** What failed is the
  profile. Four findings, all conceded, folded in below and each marked
  *(round 2)*. Two of them correct claims that had been made *to* me and that I passed
  through without testing; those are marked as such rather than quietly fixed.

The original decomposition and the original v1 framing are kept struck through. The wrong
version is part of how the right one was reached.

### Review breadth — under this ADR's own bar

A council of four local families was attempted on the design brief. **Three of the four
calls produced nothing**, and this was found only by an audit after the fact:

| critic | 2026-09-04 UTC | result |
| --- | --- | --- |
| `llama` | 10:59:22 | `RemoteDisconnected` — killed mid-request |
| `gemma` | 11:00:00 | `RemoteDisconnected` — killed mid-request |
| `qwen3:14b` | 11:01:27 | answered: FLAWED (this is round 1) |
| `glm` | 11:02:03 | **0 bytes** |

`earlyoom` SIGKILLed the model daemon at ~10:59:22Z; it stayed dead 2h17m and set no
Docker OOM flag. The calls wrote their exit status to nobody, so three empty reviews sat in
a directory looking like a council.

So **round 1 was one model and round 2 was one operator-relayed critic — both rounds are
single-source.** A foundational decision here wants at least two decorrelated families. This
ADR does not have that yet, and must not be read as if it does. It is *unverified breadth*,
not a known-wrong design: every finding that was actually returned is folded in below.

Owed before Accepted, in addition to the two open questions: **a genuine second family on
the design as written**, with the call's exit status checked rather than its output file's
existence taken as success.

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

So the link is not there to be found in the written record.

> ~~**An observation channel does not find it — it makes it.** A machine that reports "a
> unit named `keycloak.service` is active here" states a fact nobody wrote down.~~
>
> **Withdrawn *(round 2)*, and it was my sentence as much as anyone's.** It is not true of
> v1. The emitted fact is `env:<id> has_active_unit "<unit-name>"`. `env:<id>` is not tied
> to `net:host:<site>/<name>`, and a unit name is not tied to a service identity. **Both
> identity edges the join needs are still missing**, so v1 does not make the
> service→host link — it makes a *different* fact that a later, separately evidenced
> step might one day join. Saying otherwise would repeat exactly the overclaim the
> campaign has spent the day removing.

**Honest v1 scope, stated once and plainly:** a history of literal runtime artifacts
observed on one environment incarnation. Not service placement.

## Decision

> **REVISED before Accepted, 2026-09-04, on an operator proposal.** The first decomposition
> — one contract, three peer adapters (local, k8s, SSH) — is kept below struck through,
> because the reason it was wrong is the useful part. The revision splits **what to
> observe** from **how to reach it**. Assessment of the proposal, including where I push
> back on it, is in *"The split, assessed"*.

### ~~Original: one contract, three peer adapters~~

~~Define **one shared contract**, the *environment-observation profile*, implemented by
several `connector` plugins. A conforming connector answers one question — *what is
actually running in this environment* — and reports observations, never claims. ADR-21's contract is unchanged: kernel drives, adapter is a sidecar, runs
are finite and scheduled, skips are recorded, `truncated` is preserved.

~~Three implementations — `localenv_connector`, `k8s_connector`, `sshhost_connector` —
local first, Kubernetes second, SSH last.~~

**Why that was wrong:** three peer adapters would each need to know that
`systemctl list-units --type=service --state=active` answers "what runs here", and how to
parse it. That knowledge is *identical down every transport*, so it would be written three
times and the copies would drift. The same failure this campaign spent a day on, in a new
place.

### Revised: three things, not two

An earlier revision split *what to observe* from *how to reach it*. That was right and did
not go far enough — it still let Kubernetes appear in two roles. The shape is **three
things**:

**1. The k8s controller.** Kubernetes only, through the API, no shell. It answers *what
exists and where*. It is **not a transport**.

**2. The POSIX observer.** One body of knowledge about what to ask a Unix-like OS — which
units are active, what is listening, versions, mounts — and how to parse each answer. It
**owns its own transports**: local exec, SSH, `docker exec`, `kubectl exec`. That
`kubectl exec` happens to use Kubernetes credentials does not make Kubernetes a transport;
exec-into-a-pod is the POSIX observer's business, and the control plane is the
controller's.

**3. Targets.** Where the POSIX observer should go. Three possible sources: an explicit
list in configuration, the k8s controller, or Swarm's own graph — which already holds 754
site-qualified hosts from the hypervisor connector.

This is why the double-role confusion is gone: **Kubernetes never appears twice.** The
knowledge about a Unix-like OS is written once, transport-agnostic, so it cannot drift
between copies; the control plane is a separate observer with its own contract; and target
selection is a third concern that neither of them owns.

A client laptop needs no new concept: POSIX observer, local transport, itself as the
target.

### The observer reports what it *reached*, not what it was told to reach

> ~~Taking targets from the graph closes a loop, so a wrong host in the graph launders a
> data error into evidence, and the envelope must therefore record **why** a target was
> chosen.~~
>
> **Withdrawn.** That framing was wrong, and dropping it removes machinery rather than
> adding it. If the observer reaches `10.1.2.3` and that machine reports
> `keycloak.service` active, **it is a true fact about `10.1.2.3`.** The connector reported
> exactly what it saw and introduced no error at all. A target-selection ledger would have
> been provenance bookkeeping for a problem that does not exist.

The real issue is narrower. The observer knows *"the thing at address X"*. Whether that
thing is the graph node anyone **meant** is a question of **identity binding**, not of
observation quality — the same gap ADR-17 died on, and the same one round 2 named when it
said `env:` is not tied to `net:host:`.

**So the rule is: every observation carries the environment's own self-identification** —
machine-id, hostname, pod or container UID, whatever that environment can state about
itself — beside the target that was dialled. Then *"asked for A, reached something calling
itself B"* is visible at once, with no extra ledger and no new component.

Three things fall out, all free:

- **the target loop is safe** without target-provenance machinery: a mis-supplied host
  produces a correct observation about a *different* environment, and the mismatch is
  computable from the envelope alone;
- **responsibility lands where it belongs.** Supply the wrong host and that is the
  caller's error — and now it shows, rather than being absorbed silently;
- **a drift detector, for nothing.** Intended target versus self-reported identity is
  exactly the signal this campaign has spent the day trying to construct. It arrives as a
  side effect of stating identity honestly.

### Build order: one new thing at a time

The simplest thing that works, and it is deliberately smaller than the first draft's:

> **v1 = the POSIX observer, ONE transport (local), and an EXPLICIT target list in
> configuration.**

Graph-driven targets and the k8s controller come **after** that is proven, one at a time.
Introducing a new observer, a new transport and a new target source together means a
failure cannot be attributed to any of them — the same reason this campaign ablated the
three placement fixes separately, which is what turned an unattributable 12/18 into
cue +4, binding +7, precedence 0. That lesson was expensive; spending it here is free.

Order, therefore: local transport → explicit targets → a second transport (SSH, with its
own enforcement) → graph-driven targets (with the loop provenance above) → the k8s
controller as a separate observer.

### Security decomposes along the same seam

The *set of reads* is a property of the **observer** — it is a statement about what we want
to know, identical whichever way we reach the machine. *Enforcement* is a property of the
**transport**: server-side `command=` or a forced-command wrapper for SSH, RBAC for
`kubectl exec`, nothing needed for local. The first draft mixed these, putting the
allowlist in an "SSH security" section as though its contents were an SSH concern. They
are not; only its enforcement is.

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

### D1b — What an environment IS, before choosing an identifier *(round 2)*

The first draft picked identifiers before defining the thing. `/etc/machine-id`, a pod
UID and a namespace UID are not the same kind of thing: an OS installation, a running
incarnation, and a logical cluster sit at **different lifecycle levels**. Container UIDs
change on every redeployment; machine-ids survive redeployment and can be **cloned** by
imaging a VM.

So, definition first:

> An **environment** is a bounded execution context that is observed as a unit. It has two
> identity levels, and facts attach to different ones.
>
> - **Continuant** — the thing that persists across restarts and is what a fact is *about*.
>   For a VM, the OS installation. For a Kubernetes workload, the controller (Deployment /
>   StatefulSet) identity, **not** the pod.
> - **Incarnation** — the specific running instance: pod UID, container UID, boot id. An
>   attribute of the run, never the subject of a fact.

Consequences that follow, rather than being bolted on:

- observations are keyed on the **continuant**; every run records its **incarnation**, so
  a redeployment is visible as a new incarnation of the same subject rather than as a new
  subject;
- **a cloned machine-id is an identity collision, and is refused, not merged.** If the same
  machine-id is observed on two concurrently-live environments, both are quarantined and
  neither is written. This is the ADR-17 rule applied before it can bite: an identifier
  that turns out not to be unique is not an identity.
- where a continuant identifier cannot be established at all, the adapter **refuses to
  run** rather than inventing one.

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

### D2b — Observer disappearance, the case with no answer *(round 2)*

The dangerous one, and the first draft had nothing for it. A replaced or unreachable
environment **never supplies a final complete run**, so under D2's own rule its intervals
can never be authoritatively closed. Left alone, the graph accumulates facts that are
neither current nor closeable, and every one of them still reads as true.

Chosen, of leases / freshness / explicit retirement: **all three, in defined roles, and
never time-based closure.**

- **Lease.** Each environment declares an expected next-observation interval. Cheap, and
  it makes "should have reported by now" a fact rather than a judgement.
- **Freshness is the visible degradation.** When a lease lapses, facts are **not** closed.
  They stop being served as current and are rendered as *"last observed on X at T; current
  state unknown"* — the phrasing `board/todo/source-authority.md` already settled for a
  stale authoritative source.
- **Explicit retirement is the only other closer.** An operator (or a control plane
  reporting the workload deleted) retires the environment, which closes its open intervals
  with a recorded reason and a retirement instant.

**Never close on elapsed time.** Time-based closure is absence asserted without a complete
run, which is precisely what D2 forbids; a network partition would silently delete a live
estate.

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

### D3b — The observation-run envelope *(round 2, the critic's asked-for change)*

Every run carries an envelope; without it, none of D2's or D2b's rules are expressible.

| field | why it must be there |
| --- | --- |
| **environment continuant id** | what the facts are about |
| **incarnation id** | which instance produced them; a redeployment is a new incarnation, not a new subject |
| **observation class** | `active_units`, `listening_sockets`, … — closure is per class, never global |
| **declared coverage boundary** | what this run *attempted* to cover, so a narrower-than-expected run is visible rather than looking complete |
| **snapshot token** | ties pages of one logical read together, so a set assembled across pages is known to be one snapshot rather than a mix |
| **status** | `complete` / `partial` / `unsupported` — three, not two: `unsupported` is how an environment that has no systemd says so without looking empty |
| **intended target** | the address or handle actually dialled — not *why* it was chosen, just what was asked for |
| **self-reported identity** | what the reached environment says it is. **Required.** Never taken from the target entry, or the observer would be reporting what it was told rather than what it found. Intended-vs-reached mismatch is computable from these two fields alone |
| **transport** | which transport carried the run, so a transport-specific failure mode is attributable |

**The reconciliation rule, stated exactly:** absence may close a fact **only** for *that
environment*, *that observation class*, after a **completed final page** of a single
snapshot. Not across classes, not across environments, not on a partial run, and not on
`unsupported`.

`unsupported` earns its place immediately — see the internal inconsistency below.

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

## The two questions ~~no critic answered~~ — answered 2026-09-04 by two families

~~No critic answered these.~~ **They were never asked.** The two local critics intended for
(c) and (d) died mid-request when `earlyoom` SIGKILLed the model daemon, and what got
recorded instead was a conclusion about the *models* — see *Review breadth* above. Retried
on a healthy daemon, both answered, and **both returned FLAWED on both questions.**
Verdicts below; each claim was then checked against the code rather than accepted.

### (c) completeness — FLAWED, and the two families converge

`llama3.3:70b`: a `complete` from the Kubernetes observer "implies a stronger guarantee
than the same status from the POSIX observer".

`gemma4:31b`, the sharper diagnosis: *"K8s `complete` is a point-in-time snapshot; POSIX
`complete` is a temporal smear. The three-valued status is a red herring for POSIX because
it tracks **execution success** (did the command run?) rather than **state consistency** (is
this a coherent snapshot?). Downstream this leads to phantom states where the system
asserts a configuration that was never live."* Proposed fix: an `is_atomic` flag on the
envelope.

**Checked, and it is real — the defect is narrower and more specific than either said.**
Per-observation-class completeness is *necessary but not sufficient*, because the run-level
envelope oversells coherence: `Hive.Posix.Connector` derives **one `snapshot_token` from a
single `observed` timestamp for the whole run**, and then executes the classes
**sequentially**, each with its own `transport.exec`. So a token whose stated purpose is
"pages of one logical read belong to the same snapshot" is stamped across classes read
seconds apart. Nothing today reads those facts, so no wrong answer has been served — but
the envelope currently asserts an atomicity the shell path does not have. Carded:
`board/todo/observation-run-snapshot-coherence.md`. **My per-class answer to (c) was
incomplete, and this is the correction.**

### (d) drift over months — FLAWED twice, on different grounds, and both need correcting

`llama3.3:70b`: the worse mode is **fact duplication** — the old fact persists until its
validity interval expires, so two facts describe one service. *Largely already handled:*
schema v13 `edge_validity` closes a fact at its last observation per source and supersedes
on a world-level `supersession_key` with a no-overlap constraint. Not a new failure mode.

`gemma4:31b`: the worse mode is **"Configuration Blindness"** — because observers use a
*fixed list* of commands, "if a unit is renamed, the old fact closes, but the new fact is
never emitted because the observer is not configured to look for the new name. The system
becomes entirely blind to B."

**That is wrong about this design, at the level it was claimed.** Every read here
*enumerates*: `systemctl list-units --type=service --state=active --output=json`,
`ss -H -l -tun`, `docker ps`, and the Kubernetes `list` verbs. A renamed unit **is**
discovered — it appears in the next enumeration. Blindness would require per-name queries,
which this design does not use.

It is right one level up, and that is worth keeping: a new **observation class** — a kind
of thing absent from `classes()` — is never discovered, because the class list is fixed
code. So the residual hazard is *class-level* blindness, not entity-level. Much narrower
than stated, and it does not displace the drift mode already named.

**Net on (d): no confirmed worse failure mode than the one this ADR already states.** The
stated one stands, with class-level blindness added beside it.

Both questions now have two decorrelated opinions. (c) produced a real defect and a card;
(d) did not overturn the design. Kept struck-through rather than rewritten.

### The original statement of both questions

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

## v1, concretely: POSIX observer, local transport, explicit targets

Enough to build; nothing here needs an access grant.

### The inconsistency this fixes *(round 2)*

The first draft permitted a container identity **and** named `systemctl` as the only
observation. A Swarm container commonly has no systemd, so that adapter would have
observed nothing — testing neither the local environment nor the future Kubernetes shape,
while reporting an empty set that looks like a valid answer. Two changes remove it:

- **observation classes**, not one command. `active_units` is one class; a container
  reports `unsupported` for it and `complete` for the classes it can answer.
- **`unsupported` is a first-class status.** An environment that cannot answer a class
  says so, and `unsupported` **closes nothing** — distinct from `complete` with an empty
  result, which does.

### What v1 observes

| class | reads | when unsupported |
| --- | --- | --- |
| `active_units` | one `systemctl list-units --type=service --state=active --output=json` | no systemd (most containers) |
| `listening_sockets` | one `ss -H -l -tunp` | no `ss` binary |

Two classes, not one, precisely so per-class closure and `unsupported` are exercised
rather than merely specified.

### Envelope, filled in for v1

| field | v1 value |
| --- | --- |
| continuant id | **self-reported only.** `/etc/machine-id` for a host. A container that cannot state a continuant identity of its own is recorded as **incarnation-only** — never given one from the target entry, which would be the observer trusting what it was told |
| incarnation id | boot id for a host, container UID for a container |
| observation class | `active_units` \| `listening_sockets` |
| coverage boundary | "this environment, this class" — v1 never claims more |
| snapshot token | one per run; v1 is single-page per class, and the field exists so multi-page transports do not have to change the contract |
| status | per class, `complete` only when the command exited 0 **and** its output parsed |
| intended target | the literal config entry (v1: `self`) |
| self-reported identity | machine-id and hostname as the environment states them; recorded even when v1's only target is `self`, so the mismatch check is exercised before it can matter |
| transport | `local` |

### Rules

- a unit whose name does not parse is a recorded skip, never a silent drop (ADR-21);
- an environment that can self-report only an **incarnation** (a typical container) is
  recorded as incarnation-only. Its facts are explicitly **non-continuous**: they do not
  survive redeployment and no history is claimed across incarnations. That is a real
  limitation of what such an environment can say about itself, not a defect to paper over
  by borrowing an identity from configuration;
- refuse to run if the environment can state **no** identity at all, rather than inventing
  one;
- refuse and quarantine if a continuant id is already bound to another live environment —
  a cloned machine-id is a collision, not a merge;
- scope: the source scope of the `environment` source, per ADR-20;
- never: no shell pipeline, no config file reads, no process table, no network probing,
  no writes.

### What v1 does and does not test — stated, because the first draft overclaimed

*(round 2: I was told the local adapter tests the port, and I repeated it without checking.
It does not.)*

| exercised by v1 | **not** exercised by v1 |
| --- | --- |
| the observer/transport seam (one transport, but through the seam) | a second transport, and transport-specific enforcement |
| two observation classes, and per-class closure | RBAC-limited partial coverage |
| `unsupported` as distinct from empty-and-complete | multi-command partial success across classes |
| the envelope end to end, including self-reported identity | pagination and a stable snapshot across pages |
| the intended-vs-reached mismatch check (trivially, since v1's target is `self`) | that check firing on a genuinely mis-supplied target |
| continuant/incarnation separation on redeploy | observer disappearance and retirement over real elapsed time |

The right-hand column is the argument for the build order above, not a gap to apologise
for: each item arrives with the increment that first needs it.

Test before trusting: a fixture environment with a known unit set, a container fixture
that must report `unsupported` rather than empty, and a partial-run fixture that must
close nothing — the same shape as the learner-eval fixtures, and for the same reason.

## What must be true before this is Accepted

1. **The profile survives, the shape changed.** Round 2 confirmed the connector-kind
   decision, so *"a profile, not a new port kind"* stands. Under the three-part shape the
   profile is narrower than first written: it governs the **observation-run envelope** and
   the reconciliation rule, which the POSIX observer and the k8s controller both obey.
   Transports are not part of it — they are a property of one observer.
2. A decorrelated critic on **(c)** and **(d)**, still unanswered by round 1 and not put to
   round 2. Note (c) has largely dissolved: with the observer owning its transports, "one
   contract for a shell prober and an API prober" is no longer the question — they are two
   observers sharing only an envelope.
3. Agreement that v1's honest scope — *a history of literal runtime artifacts on one
   environment incarnation* — is worth building, given it does **not** close the
   service→host gap that motivated the campaign. Both identity edges (`env:`→`net:host:`,
   unit→service) remain missing and are out of v1's scope.
4. The D1 precondition check for the environment↔inventory tie, named and scheduled as a
   read the operator authorises — not an assumption this ADR may make.

## What this ADR still does not answer

Written down rather than left implicit, because three rounds of review have each found one:

- **the unit→service edge.** `has_active_unit "keycloak.service"` is not "runs Keycloak",
  and nothing here proposes how that inference would ever be evidenced.
- **the `env:`→`net:host:` edge.** Blocked on the D1 precondition; if that check fails,
  observation and inventory stay two disconnected keyspaces and the campaign's measured
  gap is untouched.
- **(d) what breaks after months.** My guess remains unit-name drift on migration: the
  graph faithfully records one thing stopping and another starting and nobody is told they
  are the same service. D1b's continuant/incarnation split addresses the *environment*
  version of this and does nothing for the *unit* version.
