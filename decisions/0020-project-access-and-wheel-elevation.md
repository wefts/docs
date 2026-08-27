# ADR-20: Project Access And Wheel Elevation

## Status

Accepted (2026-08-27).

Supersedes part of ADR-18 and ADR-19:

- ADR-18's per-source scope substrate remains, but direct `group -> src:*` grants are superseded
  by `Project membership -> Project Source -> effective source scopes`.
- ADR-19's `Superuser` / `Admins` / `Everyone` group model and standing group-derived
  `superadmin` are superseded by `Wheel` / `Admins` / `Staff` and time-boxed elevation.

Preserves ADR-16's substrate: signed actor assertions, kernel-derived identity/scopes/caps,
`scope × owner`, default-deny, single access gate, and audited break-glass through the same
filtered path.

## Record Completeness

Complete. Council record:
`board/research/project-access-blackboard.md`.

## Context

The access model grew through implementation pressure: ADR-16 established the no-leak substrate,
ADR-18 evolved coarse `group` visibility into per-source `src:*` scopes, and ADR-19 tightened
admin authority around `Superuser` / `Admins` / `Everyone`.

That model works, but the product noun is wrong. Users do not think in source scopes; they think in
workspaces, shared knowledge areas, private material, and invited people. Group-granted source
scopes also blur two planes:

- data access: what knowledge an actor can see;
- administration: what system operations an actor may perform.

Projects give the system a single user-facing sharing container while preserving the proven kernel
gate.

## Decision

1. **Projects are the sole data-access container.** A Project owns Sources/Connectors. Membership
   in a Project grants visibility to that Project's source scopes. No data-access grant exists
   outside explicit audited Project membership.
2. **Sources are stable security coordinates.** A connector's human label is not the access key.
   The graph scope is a stable source coordinate such as `src:<source_uuid>`, unique per source
   instance. Two Confluence connectors in two Projects must not collapse into one global
   `src:confluence` scope.
3. **Groups do not grant source visibility directly.** A group may be a Project member; users in
   the group then inherit that Project's source scopes through Project membership.
4. **Direct user Project membership is allowed.** This is explicit audited membership, not a
   hidden per-user grant outside the Project model.
5. **Project visibility words are product classifications, not a second access system.**
   `personal`, `shared`, and `public` describe Project shape. The access decision still derives
   from Project membership, source scopes, and the owner axis. `own` is not Project visibility; it
   means `owner = actor`.
6. **`public` is intentional baseline material.** It is not computed from "all users are members"
   and cannot appear or disappear as an accidental result of membership churn.
7. **The fixed groups are `Wheel`, `Admins`, and `Staff`.**
   - `Wheel` is local-only and grants the right to request elevation.
   - `Admins` grants ordinary administration capabilities inside a hard boundary.
   - `Staff` is the default internal cohort and grants no role by itself.
8. **Roles and data visibility stay separate.** Roles confer capabilities; Project membership
   confers source visibility. No role makes private or out-of-project data visible by itself.
9. **There is no standing `superadmin` role.** `superadmin` exists only as an active elevation
   session: local `Wheel` member, fresh re-authentication, required reason, time-boxed expiry, and
   audit before capability takes effect.
10. **Data break-glass remains per-operation.** Elevation does not create an ambient root read
    path. Cross-user support reads still require target, reason, request id, effective lens, and
    audit before data is returned.
11. **Admin boundaries are hard.** `Admins` cannot modify `Wheel`, role bindings,
    auth/elevation/audit controls, publicness, or any path that would let them self-escalate.
12. **Service/root context is not a user lens.** Enrichment and maintenance may use service
    authority, but outputs inherit input source/owner coordinates. Scope widening is never
    implicit.

The readable canonical model is [../architecture/access-model.md](../architecture/access-model.md).

## Consequences

- The product gains a clear sharing noun: users join and share Projects, not raw scopes.
- The proven kernel no-leak predicate survives unchanged:

  ```text
  visible ⇔ scope ∈ actor.effective_scopes
         AND (owner IS NULL OR owner = actor.id)
  ```

- The source of effective scopes moves from direct group grants to Project membership.
- The admin model becomes smaller and closer to common Unix/RBAC practice: `Wheel` may elevate,
  `Admins` administer, `Staff` is a cohort.
- Current implementation and docs that assume `Superuser` / `Everyone` or direct
  `group -> src:*` grants need migration.
- Existing graph rows with human-readable `src:<name>` scopes need a migration to stable
  source identifiers before two same-kind connectors can safely coexist.
- All user-facing read surfaces must be re-proved under project-derived scopes: ask, search,
  traversal, dashboard projections, export, world-map serve, ready answers, and admin views.
- The model avoids a generic ACL engine for now. The forcing condition for a richer ACL or
  `text[]` visibility model is real union visibility: one row must be visible through multiple
  independent Project/source memberships.

## Alternatives

- **Keep direct group-to-source grants.** Rejected: workable for the first cohort, but it makes
  groups carry both administration and data-access meaning and lacks a user-facing sharing object.
- **Use graph-per-project or graph-per-tenant.** Rejected: loses the shared memory and
  consolidation benefits that justify the graph substrate.
- **Make Project visibility depend on membership cardinality.** Rejected as a security primitive:
  group membership cardinality is indirect, mutable, and easy to misread. Keep the words as product
  classifications; keep authz explicit.
- **Keep standing group-derived `superadmin`.** Rejected: it turns break-glass into a permanent
  role. Elevation is cheaper to audit and easier to reason about.
