# ADR-18 (workspace): Per-source scope + first-class groups (ADR-16 access evolution)

## Status

**Proposed (2026-07-08).** Evolves **ADR-16** (users / identity / privacy) — it does NOT
reopen the identity anchor or the per-user-conversation privacy decisions; it evolves ADR-16's
**access mechanism** (the coarse `public`/`group`/`private` scope) into per-source scopes and
makes groups first-class. **Council-gated before execution** — the forks in "Open forks" below
must be closed by a 2-family blackboard (codex + gemini) before any migration. Spec →
`swarm/docs/design/per-source-scope-authz.md`; cards → `board/todo/` (`ps-*`); planning pass
recorded in `board/journal.md`. Operator model decision: memory
`authz-model-roles-admin-groups-access`.

> NB numbering: a **workspace** ADR (spans kernel scope substrate + channel login + hive env),
> distinct from the swarm-local sequence.

## Record Completeness

Direction + the decided model are settled (operator, 2026-07-08). The load-bearing *mechanism*
forks (scope ordering, single-column vs ACL, multi-origin nodes, migration key) are named here
and resolved by the execution council — this is the planning pass, not the final mechanism.

## Context

- The admin console is going from admins-only to an **`Everyone` (non-admin) cohort** so the
  MVP can be shown to people outside the DSI team. That is the committed goal — the work below
  is the path to it, not something waiting for a trigger.
- Today visibility is a single fixed enum `private`/`group`/`public` on every node + edge
  (`Swarm.Graph.Contract`, `@scope_rank`), and the whole no-leak proof rests on it (edge + both
  endpoints every hop, RLS, the world-map who/network serve family).
- The coarse model is **all-or-nothing** for the `group` cohort: a non-admin either sees the
  entire group-scoped corpus (wiki AND confluence AND ldap directory AND the network/IaC map)
  or nothing. It cannot express the decided baseline **`Everyone` = wiki + ldap, NOT
  confluence / network-map / IaC**.
- Decided model (operator): ROLES = administration only (user/admin/superadmin), never grow
  with connectors; ACCESS = per-source scope, each connector registers `src:<name>`,
  default-deny; GROUPS = cohort bundles granting a SET of source-scopes AND/OR a role;
  connectors are NEVER roles.

## Decision (direction; mechanism forks below)

1. **Scope value space evolves** from `{private, group, public}` to
   `{private, public, src:<name>, …}`. A node/edge's scope is **derived from its ingest
   origin** — `edge_provenance.origin` already carries `wiki:`/`confluence:`/`iac:<repo>`/
   `ldap:directory`, so the source is known; that origin is the **migration key** and the
   ongoing derivation key. `public` stays the **universal baseline** (the authenticated⇒public
   floor that two prior regressions destroyed — preserved explicitly). `private` stays
   **ungrantable** (the `user` person-node pin + DB CHECK — unchanged from ADR-16).
2. **A viewer's effective scopes = `["public"] ∪ (src-scopes granted via their groups)`.**
   Default-deny: no group ⇒ only `public`.
3. **Groups become first-class entities** (`{id, name, description}`) with granted src-scopes
   AND an optional **group→role** binding (roles today are per-user `role_grant` only — a group
   cannot confer a role yet; the model requires it). `group_scope_map` holds `src:<name>`
   values; `roles_for/caps_for` become `direct role_grant ∪ group→role`.
4. **SSO group → our-group mapping** is config-driven (which claim carries groups/roles;
   incoming-group → our-group table), evolving the env `GROUP_SCOPE_MAP`. Unmapped ⇒ nothing.
5. **Initial config:** Superuser → all `src:*` + superadmin; Admins → all `src:*` + admin;
   Everyone → `src:wiki` + `src:ldap` + user (NOT confluence / network / IaC).

## Open forks (the execution council MUST close these before code)

- **F1 — scope ordering.** `@scope_rank` assumes a total order (`private<group<public`) and the
  ingest path clamps `min(source, group)`. `src:*` scopes are a **set, not a rank**. Proposal:
  a node carries exactly one `src` scope (its origin) + `public` is the floor; the "clamp"
  becomes set-membership ("is the viewer granted this src?"), and `private`/`public` keep their
  rank extremes. Council confirms this collapses cleanly across all ~10 predicate sites.
- **F2 — single column vs ACL.** Keep the single `visibility_scope` string (open namespace,
  least migration) vs a separate node↔scope ACL table (multi-scope, heavier). Recommend the
  single column unless a node legitimately needs to belong to >1 source.
- **F3 — multi-origin nodes.** A node corroborated by wiki AND iac: which `src`? (union of the
  contributing sources → visible to anyone granted ANY of them, vs most-restrictive.) Ties to
  the corroboration/ghost-purge path — must not let corroboration silently widen visibility.
- **F4 — public-baseline + regression guard.** Re-prove the authenticated⇒public baseline and
  the group→scope derivation survive (both broke on the ADR-16 cutover — positive controls
  required, not just "0 hits").

## Consequences

- Enables the `Everyone` cohort → the MVP-external. Every scope predicate site is touched;
  RLS re-proof required; the migration rewrites every current `group` node's scope. Connectors
  must register their `src` at the ingest boundary. Reversible via snapshot + a down-migration.
- The world-map serve family (who/network) inherits per-source scoping for free once the
  predicate change lands — and `world-map-serve-governance`'s `policy_filter` becomes the
  natural place to enforce it answer-side.

## Alternatives rejected

- **Keep coarse + duplicate the corpus per cohort** — unmaintainable, and re-ingests leak.
- **A role per source** — operator explicitly rejected (connectors are never roles; roles stay
  administration-only).
- **Separate full ACL engine now** — over-built for a handful of sources + a 3-group config;
  revisit only if per-node multi-source membership becomes real (F2).
