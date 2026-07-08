# ADR-18 (workspace): Per-source scope + first-class groups (ADR-16 access evolution)

## Status

**Accepted (2026-07-08).** Evolves **ADR-16** (users / identity / privacy) — it does NOT
reopen the identity anchor or the per-user-conversation privacy decisions; it evolves ADR-16's
**access mechanism** (the coarse `public`/`group`/`private` scope) into per-source scopes and
makes groups first-class. The mechanism forks F1-F4 (+ SSO-store, group-id) were closed by a
2-family blackboard (codex gpt-5.5 + gemini 3.1-pro, strong convergence) — see
`board/research/per-source-scope-blackboard.md` and "Forks — RESOLVED" below. Spec →
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

## Forks — RESOLVED (council 2026-07-08; full record + census in the blackboard)

- **F1 — scope ordering → RESOLVED: lattice, incomparable `src:*`.** `private`=⊥, `public`=⊤,
  each `src:*` an orthogonal mid-band tag. Census fact: all ~11 READ sites are ALREADY pure
  set-membership; rank is used only at write. The write clamp is unified to ONE rule — the lattice
  **greatest-lower-bound** of an edge's endpoints (replaces both duplicate `@scope_rank` maps,
  `Ingest.narrowest/2`, and `Contract.check_visibility`): `GLB(public,public)=public`,
  `GLB(src:X,public)=src:X`, `GLB(src:X,src:X)=src:X`, `GLB(src:A,src:B|A≠B)=private`,
  `GLB(private,_)=private`. Degrades to today's behavior on the `{private,public}` subset.
- **F2 — single column vs ACL → RESOLVED: single `visibility_scope` column.** Shape
  `^(private|public|src:[a-z0-9_-]+)$` (alter the DB CHECK from the fixed list). Forcing condition
  to upgrade to a `text[]`/ACL model = when UNION visibility is a real need (an ldap-only viewer
  must see an ldap entity that wiki ingested first) — NOT now.
- **F3 — multi-origin nodes → RESOLVED: first-writer-wins node scope (never widened).**
  Corroboration/lineage is orthogonal to scope — a later origin NEVER rewrites a node's scope
  (`upsert_node ON CONFLICT` already doesn't; make it an intentional, tested security invariant).
  Cross-scope merge stays REFUSED. A cross-src EDGE → GLB = `private` (safe; the "invisible
  cross-source edge" is the accepted cost, see below). **ps-2 must first MEASURE the existing
  cross-src edge count** (synonymy/ER edges spanning sources) — a nonzero count is a regression to
  weigh before the migration flips them to `private`.
- **F4 — public-baseline + regression guard → RESOLVED: positive-control matrix.** Baseline is
  structurally guaranteed by `scopes = ["public" | group_scopes] |> Enum.uniq()`. ps-5 ship gate =
  exact set-equality persona tests (no-group ⇒ `["public"]`; wiki+ldap persona sees `public∪wiki∪
  ldap` and EXACTLY 0 confluence/iac/network; unmapped SSO group grants nothing), on the REAL serve
  path with real entail (not stub — memory `verify-real-serve-path-not-stub-entail`).
- **Sub-A SSO mapper store → RESOLVED: kernel table** (`sso_group_map`; authz is kernel-authoritative,
  migratable, audited — not channel config). **Sub-B group id → RESOLVED: UUID pk + mutable `name`
  (+ optional unique slug)**; SSO/grant keys reference the uuid, so an upstream rename is a metadata
  update, never a cascading grant delete.

**Accepted cost (the #1 risk, consciously bounded):** cross-source edges clamp to `private` →
invisible even to a viewer holding BOTH srcs (the graph fractures at src boundaries for shared
traversal). Acceptable because the current cohort need (`Everyone` = wiki+ldap NODE visibility)
needs no cross-src EDGE traversal; the E4 wiki↔ldap uid-join knowledge-links are the deferred
trigger to upgrade to a `text[]` scope. ps-2 also audits the `activity` predicate (node.scope OR
edge.scope) so an edge-scope-alone path can't surface a cross-endpoint relationship.

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
