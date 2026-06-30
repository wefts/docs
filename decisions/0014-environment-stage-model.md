# ADR-14: Environment & Stage Model

## Status

Accepted

## Record Completeness

Complete

## Context

The workspace had no defined stage taxonomy, and the database names had drifted
from their roles — which made "what runs where" recurrently unclear:

- There are **no external / SLA users yet** — single operator, local-first.
- The two most official-sounding names are the most misleading. `swarm_prod` is
  **not** production: it is the one live instance on real preproduction data
  (`docs/STATE.md` already conceded «"prod" = preproduction»). `swarm_dev` is
  **not** a friendly dev sandbox: it is the value `SWARM_DB_NAME` silently falls
  back to when unset, which forced the kernel to invent a "conditional-prod"
  guard concept just to stop tooling from mutating it by accident.
- The classic `dev → stage → uat → pp → prod` pipeline assumes a release train
  and external users. With neither, most of those tiers would sit empty, and an
  empty tier adds fog, not clarity.
- Going public **is** planned (a public domain + real SSO + invited users on a
  small Kubernetes substrate — see `board/ideas/go-public-deployment.md`). At
  that point a real production tier finally earns its name.

## Decision

Adopt a **role-based** model with exactly the tiers that have a function:

| Role | What it is | Today's DB / substrate | Guarantee |
|---|---|---|---|
| **test** | ephemeral, hermetic, CI | `swarm_test` | wiped every run |
| **sandbox** | disposable experiments, loop shakedowns, risky mutating runs | `swarm_slice`, `swarm_shadow`, clones | throwaway |
| **staging** | the single **internal live** instance: real data, internal / limited users, fast iteration | `swarm_prod` DB, docker-compose | snapshot before mutating; no silent writes |
| **prod** | public domain, real SSO, invited external users | **does not exist yet** — planned on k3s | (future) |

Two governing principles:

1. **A tier exists only when it has a function.** `uat`, `pp`, and similar stay
   unborn until a concrete need appears. Going public adds exactly **one** real
   tier (prod), and re-roles today's live instance into **staging**.
2. **A clone is an instrument, not a tier.** A per-run disposable copy of the
   live DB (for a long mutating run — e.g. the multi-day equilibrium loop) lives
   under *sandbox*; it is created, used, and discarded. It does not earn a name.

## Consequences

- The current internal console (the Spark instance) is **staging**, not prod.
  "Going public" introduces prod on a separate substrate; staging stays on
  docker-compose until/unless substrate divergence forces a migration (the
  phasing and the deployment design are in `board/ideas/go-public-deployment.md`).
- **Names should stop lying.** The `swarm_dev` silent default is a footgun:
  unset `SWARM_DB_NAME` should be a hard error everywhere (the loop harness
  already refuses it; the kernel boot and compose should too). Once there is no
  silent default, the "conditional-prod" concept disappears. Tracked in
  `board/todo/stage-model-normalization.md`.
- A full DB rename (`swarm_prod` → a name that tells the truth) is **deferred but
  bounded by a hard trigger**: it must complete **before a real prod tier exists**.
  Never run prod next to a staging DB literally named `swarm_prod` — at that point
  the word "prod" would point at the *non*-prod, which is the highest-stakes
  footgun (a destructive command, or a safe-on-staging assumption, lands on the
  wrong target). Today the risk is low (single operator holding context + the
  `current_database()` guards), and the rename ripples through `.env`/scripts/
  `STATE`, so it waits — but the deadline is the **go-public boundary**, not
  "someday". Deferral is bounded, not open-ended; the cost of the misnomer
  accrues monotonically (every new reference cements it).
- Docs that reference stages (the `STATE` operational note, the runbook) should
  normalize to this vocabulary.

## Alternatives

- **Classic `dev/stage/uat/pp/prod`** — rejected: it imports a release-train
  ceremony for users and a pipeline we do not have; the empty tiers were the
  source of the fog, not a cure for it.
- **One environment for everything (keep running on `swarm_prod`)** — rejected:
  that is precisely what produced the confusion and the risky-experiment problem;
  experiments and the live instance need separation (the snapshot/clone
  discipline depends on it).
- **Rename the databases now** — deferred (see Consequences): cost outweighs the
  benefit at this stage.
