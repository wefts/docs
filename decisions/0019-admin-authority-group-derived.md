# ADR-0019 — Admin authority is group-derived; fixed groups; superadmin only via Superuser

## Status

Accepted (2026-07-10, operator-approved after council). Evolves ADR-0016
(users/identity/privacy) and ADR-0018 (per-source scope + first-class groups).

## Context

ADR-0016 gave us roles → capabilities (default-deny) + RLS; ADR-0018 added per-source
`src:*` scopes, first-class groups, group→role, and SSO claim mappers. The admin console
(hive) was realigned to present this model. But the **runtime still contradicts it**:

- Admin authority is decided by `Principal.is_groot`, derived from a **Keycloak token role
  `groot`** or a local-credential flag — not from kernel group membership.
- `ManageAccess` lets **`superadmin` (and `admin`) be granted directly to a user**, and
  `ManageGroup` lets **`superadmin` be bound to any group**.
- Scopes can still originate from the channel env (`GROUP_SCOPE_MAP`) / per-credential rows.
- There is no way to list a group's **members** (the group page shows a count only).

So "roles/connectors hang on groups" is drawn in the UI but is not the enforced source of
truth. The operator hardened the model (2026-07-09/10, memory `authz-model-roles-admin-groups-access`).

## Decision

1. **Fixed group set.** Exactly `Superuser`, `Admins`, `Everyone`. No arbitrary group
   lifecycle in normal operation (the console exposes only these three).
2. **Roles attach to groups, never to a person.** No per-user role grant exists. `ManageAccess`
   `GRANT_ROLE`/`REVOKE_ROLE` (user-targeted) is **removed/rejected**; a member's role is
   derived from the roles their groups confer.
3. **`superadmin` is bindable only to the `Superuser` group.** The kernel rejects a
   `GROUP_SET_ROLE superadmin` for any group other than `superuser`, and rejects any direct
   user superadmin grant. `Admins` confers `admin`; `Everyone` confers no elevated role.
4. **`Superuser` membership is local-provider only.** The kernel rejects adding a non-`local`
   user to `superuser`; an SSO group mapping may not target `superuser`.
5. **Admin authority is kernel-derived.** Effective caps come from group→role→caps
   (ADR-0016 machinery). The Keycloak token role `groot` and the local-cred flag are **retired
   as authority paths**; the channel's `is_groot` becomes a *reflection* of kernel-derived caps
   (e.g. a `manage_*`/superadmin cap), not an independent source. Bootstrap: `groot` is a local
   user who is a member of `Superuser`.
6. **IdP realm roles are diagnostic only** — never an authorization input. Group membership
   (via the SSO group→our-group map) is the only SSO-driven authority.
7. **New read RPC: group member listing** (`GetGroup` / `ListGroupMembers`, any admin cap) so
   the group detail page shows members (login + provider), replacing the "pending" placeholder.
8. **No external IdP identity may be linked to a `Superuser` member** (council/gemini): a local
   Superuser account that links an SSO subject would hand the IdP an indirect superadmin path.
   The kernel rejects an `identity_link` whose user is in `Superuser` for any non-`local` provider.
9. **`admin` capability boundary** (council/gemini second-order): since `Admins` *is* SSO-mappable,
   a compromised IdP can mint `admin`s. `admin` must therefore categorically **exclude** modifying
   the `Superuser` group, local-provider/auth-provider config, or any role binding — i.e. no
   `admin`→`superadmin` self-escalation and no disabling of local break-glass. (`admin` already
   lacks role-grant caps in ADR-0016; this ADR keeps role binding superadmin-only and adds the
   Superuser-group/auth-config carve-outs.)

## Consequences

- The console can finally *enforce* (not just depict) the model; the hive `is_groot` gate flips
  to a kernel-caps check once (5) lands.
- **Bootstrap safety — ONE atomic idempotent migration (council/codex, the load-bearing fix).**
  The authority source must switch atomically or we get either lockout or a lingering bypass.
  In a single migration/rollout: create/verify `Superuser`; bind `superadmin` **only** there; ensure
  `groot` is an active `local` member; **assert `count(active local users with group-derived
  superadmin) >= 1`**; only THEN remove/disable the direct `role_grant` authority path and start
  rejecting `GRANT_ROLE`/`REVOKE_ROLE` on the wire. codex confirmed against the code that today
  `seed_superadmin/1` writes a *direct* superadmin grant, `caps_for` still unions direct
  `role_grant` with `group_role`, and the RPC still accepts user-targeted role ops — so this is
  new enforcement machinery, not policy text or a UI change.
- **Fresh boot (council/gemini):** the seed path (`seed_superadmin`) must itself confer superadmin
  via `Superuser` membership (not a direct grant), or a clean deploy can't bootstrap once the
  direct path is retired.
- Removing user-targeted role ops is a wire-contract change (ADR-0016 `AccessOp`); the RPC stays
  but those ops become `BAD_REQUEST`/no-op, audited. Channel already dropped the UI (Track A).
- Everyone-cohort no-leak (ADR-0018 ps-5) is unaffected; this only tightens who is admin.

## Alternatives considered

- *Keep token-role `groot` as authority:* rejected — it lets the IdP mint admins outside the
  kernel's control, contradicting ADR-0016 D9 (authority is kernel-derived, not token-asserted).
- *Allow superadmin on any group / on users:* rejected by the operator — superadmin is the
  break-glass tier and must be a small, local, auditable set (the Superuser group).

## Council (2026-07-10)

Two decorrelated critics, both **SOUND-WITH-CAVEATS**:

- **codex** (grounded in the kernel code): the model is sound but "not achievable by policy text
  alone" — `seed_superadmin/1` writes a direct superadmin grant, `caps_for` unions direct
  `role_grant` with `group_role`, and `ManageAccess GRANT_ROLE/REVOKE_ROLE` + the proto still
  accept user-targeted role ops; hive `is_groot`/`_require_groot` gate on the IdP realm role +
  local SQLite flag, an independent authority path until flipped. Single most important fix: the
  atomic migration above (assert group-derived superadmin ≥1 BEFORE removing the direct grant).
- **gemini**: endorsed centralizing authority; caveats folded in as Decision (8) no-IdP-link for
  Superuser, (9) `admin` capability boundary, and the fresh-boot seed note.

Verdict: **accept the ADR with these folded in.** All caveats are now in Decision (8)/(9) +
Consequences (atomic migration, fresh boot). No critic found a reason to reject the direction.

## Scope / sequencing

Kernel (swarm), council-gated: (a) member-list RPC; (b) reject superadmin off-Superuser + reject
direct user role grants; (c) Superuser local-only guard; (d) group-derived caps + bootstrap
migration. Then hive: flip `is_groot` → kernel caps; wire the group members list. Data (Track C)
already conforms (groot ∈ Superuser; fixed 3 groups; SSO map live).
