# ADR-16 (workspace): Users — Identity, Access, and Per-User Privacy

## Status

**Proposed — both forks now RESOLVED by a 5-source council (2026-07-01); pending operator
sign-off to flip to Accepted, then spec + cards.** Item 2 of the post-migration trio;
graduates `board/ideas/users-identity-privacy.md`.

> NB numbering: this is a **workspace** ADR (cross-cutting invariant, spans kernel +
> channel), distinct from the swarm-local sequence (whose 0016 is pg_search).

## Record Completeness

Complete — direction + both forks resolved by council; mechanism detail goes to the spec.

## Context

- Moving from two people toward an **invited cohort**. Current login (web_channel P1) is
  a scaffold: channel-side OIDC + a local pbkdf2 credential store + a `groot` admin; the
  kernel is the scope *authority* but does **not** own user accounts.
- Swarm is a **personal work assistant, not ChatGPT** → conversations are **private per
  user by default**.
- Elaborates data-foundation Principle 5 (users-as-data) and **extends the no-leak
  invariant to conversations**.

## Decision (settled)

1. **Identity anchor = surrogate `UUIDv7`.** Attributes (mutable, never the key): `login`
   (= the IdP **uid** — Smile `penta`, 3–8 chars — the login handle *and* the match key;
   maps to OIDC `uid`/`preferred_username`), `emails[]` (verifiable), `first_name`/`last_name`
   (OIDC `given_name`/`family_name`), `nickname` (alias dropped — same thing). The
   **SSO claim→field mapping is explicit and config-driven** (login/email/names/groups mirror
   SSO fields so local + SSO users are one shape; configurable because IdPs vary). A **rich
   profile** — freeform bio + structured facts (`based_in` / `interested_in` / `works_on` /
   projects) — lives on the **person-node** (graph, P5; human-entered or enrichment-learned →
   item 3), **not** the auth record. Roles beyond `is_admin` are deferred (Smile barely uses them).
2. **Login by `login`** (like Smile SSO), not email. **Local auth** (pbkdf2) + **create
   users without SSO**.
3. **SSO = JIT provision**, matched on the IdP **stable `sub`** (not email); **account-
   linking** so an SSO login and a local login resolve to **one** uuid. Built against the
   **local Keycloak**; the `sso.smile.eu` swap is a deferred operator/go-public step.
4. **Two visibility axes** (not one):
   `visible ⇔ (scope ∈ viewer.scopes) AND (owner IS NULL OR owner = viewer)`.
   Corpus docs are **scoped** (no owner); conversations are **owned**. **group → scope
   mapping** is config-driven, kernel-enforced, **default-deny**.
5. **Per-user conversation privacy = a NEW invariant of the no-leak class.** A user's
   conversations are private to them by default; **enforced**, not merely UI-hidden;
   adversarially tested with the same rigor as scope no-leak (list / search / cursor /
   neighborhood / activity / error — no path leaks another user's conversation).
6. **Admin support-read = impersonation through the SAME filtered path + break-glass —
   NOT a separate all-rows query** (council correction). Admin *assumes the target user's
   view* via the normal owner predicate (sees exactly what the user sees, no bypass),
   gated by an **explicit, time-boxed, reason-required** elevation, with an **immutable
   audit row written BEFORE data is returned** (actor, target, ids/scope, reason,
   request-id, decision). Never by default; never an `admin=true` flag on a normal read
   (that *is* the backdoor failure mode). (Operator: support-reading is legitimate for a
   work assistant — but break-glass + audited.)
7. **Roles = capabilities, source-agnostic, default-deny** (replaces the coarse `is_admin`
   bool — the council flagged that too). Two tiers:
   - **admin** — `manage_access` (grant/revoke access to **shared resources**: the corp
     wiki / Confluence today, user-created KBs later) + `invite_users` (create local users).
     **Admins do NOT read others' conversations.**
   - **superadmin** — **all** capabilities, incl. `read_any_conversation` (the break-glass
     audited path of Decision 6). A **local** account whose id is a **normal `UUIDv7` (same
     scheme as everyone) but a recognizable / vanity value** — the *root / uid-0* feel
     (memorable, not a special sentinel type), seeded at bootstrap. `groot` is just our name
     — the **role** is what matters; rename freely, and keep the literal name out of
     `swarm/` + public docs.
   A role is conferred by **group→role mapping** (SSO or local group) **or** a **direct
   grant** — local / SSO / group confer roles identically.
8. **Build the full model at once** (operator: surfaces the real advantages + problems),
   not phased.
9. **The kernel VERIFIES the forwarded actor identity — it does not trust it** (council,
   load-bearing). The channel forwards a **signed** actor assertion (a JWT the kernel
   verifies, or mTLS), and the kernel derives the effective `{uuid, scopes, owner}` from
   it. Security-bearing paths (the conversation owner-check; ideally scope grants) never
   trust a plaintext `viewer`/`scopes` field. This revisits ADR-7's opaque-*trusted*
   `viewer`: today the kernel trusts channel-asserted scopes — a *nominal* boundary a
   channel bug / stale session / confused-deputy can spoof. On the single box this is a
   cheap shared-secret HMAC/JWT; it is what makes the kernel the **real** sole authority.
10. **Access grants are kernel-owned, admin-mutable, and audited.** Group memberships +
    scope grants live in the kernel (the authz authority) and change only via audited admin
    RPCs (gated by `manage_access`), not only static config (config may seed defaults). Every
    grant / revoke / invite writes an audit row; default-deny throughout. This makes the
    group→scope map **runtime-manageable** (admins add/revoke access to shared resources),
    not just deployment config.

## Forks — resolved by council (2026-07-01)

**A → hybrid A1, MINIMAL.** The **kernel** owns the minimal *authorization* record —
`uuid` + login + emails + group/scope grants + `is_admin` + identity-links + **conversation
ownership** — provisioned **JIT** from token claims (claims are the source of truth;
idempotent upsert on login). The **channel** owns **authentication only** (password, OIDC,
session, cookies). It is **not** a full identity service in the kernel — "an authz-enforcer
that happens to persist ownership" (web). The person is **also** projected as a graph node
on the same uuid for facts (item 3); password hashes / SSO subjects **never** enter graph
edges. (Repo: ~500–700 LOC, Core proto unchanged — `viewer` is already opaque.)

**B → B1, STRUCTURAL.** Conversations are a **kernel-owned aux entity** — reuse the proven
`Swarm.Deliberation` pattern (already a kernel aux table with owner + scope-re-auth + an
opaque handle). Enforce owner-only at **one data-access choke point** that injects
`owner = verified-subject` (never a caller-supplied owner), backed by **Postgres RLS** as
the belt-and-suspenders net so a future new path / export / search cannot escape the DB
policy. Deny-by-default, UUID ids, **404-not-403** (no existence oracle). Complies with hive
ADR-1 (aux table + RPC; channel never reads the DB). (Repo: ~800–1200 LOC + 2 RPCs.)

## Council (2026-07-01)

5-source blackboard (`tmp/notes/blackboard-users-identity.md`): KS-A architect · KS-B
repo-Explore · KS-C web prior-art · KS-D codex (gpt-5.5) · KS-E llama3.3:70b. **Convergent:**
A = hybrid-minimal, B = B1. It **corrected the architect's first pass in three places
(adopted):**

1. The "kernel *enforces* but *trusts* channel-supplied `{viewer, scopes}`" middle is
   **unsound** — it makes the channel the authority again; **verify cryptographically**
   (Decision 9), don't trust a plaintext field. (This is the crux — where no-leak "lives or
   dies", web.)
2. Admin cross-user read is **impersonation-through-the-same-path + break-glass**, not a
   separate all-rows RPC (that *is* the backdoor). (Decision 6.)
3. Make isolation **structural** — single choke-point + RLS — not per-handler discipline
   ("airtight as a property of the architecture, not of remembering to check").

**Model gaps to carry into the spec:** `conversation.owner_id NOT NULL`;
`message.author_user_id` (author ≠ owner); `user.status` + `last_login_at`;
`identity_link.verified_at` + uniqueness; `email.verified_at`/primary; group-membership
provenance; `conversation.visibility` (private default; shared/team later) + `deleted_at`/
retention; full `admin_access_audit` fields; a **service/agent identity** model (background
jobs / the enrichment loop / indexers / backups / MCP must be authorized too — enrichment
must respect conversation ownership); the **search + export/backup paths** must apply the
owner predicate ("cannot read by any path" fails there first); a **person-node leak rule**
(chat-derived facts projected to the graph must not surface to scoped corpus reads); a
session/token entity.

## Consequences

- If forks A/B land kernel-side, the **kernel's surface grows** (identity/ownership) —
  weigh against the microkernel principle in the council.
- **Migration without lockout:** existing local users + `groot` + the SSO test users must
  map onto the uuid model without anyone losing access — a required build step.
- The **person-as-node** projection is the first concrete **users-as-data** (P5) and feeds
  item 3's world-map.
- `board/todo/bm25-index-hardening` gates broadening the cohort (pre-existing).
- Routing: identity/ownership/enforcement → `swarm/` (kernel); login/session/SSO/admin
  page + group→scope config → `hive/`; the invariant + model → this workspace ADR.

## Alternatives

- **Channel-owned identity (A2, status quo extended)** — REJECTED by council: a channel
  that asserts `{viewer, scopes}` the kernel merely trusts is not a real authority boundary
  (spoofable by a channel bug / stale session), and it splits the privacy boundary.
- **Channel-side conversations (B2)** — REJECTED by council: the channel has direct storage
  access, so owner-only is not airtight "by any path"; two visibility systems (corpus in
  kernel, chats in channel) is the split-brain the invariant forbids.
- **Person as a pure graph node (no auth table)** — clean "all users-as-data", but
  conflates security-sensitive auth with public-ish graph knowledge. Rejected for the auth
  layer; kept for the facts layer.
- **Phased rollout** — rejected by the operator (build all at once to surface real problems).
