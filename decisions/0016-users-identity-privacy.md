# ADR-16 (workspace): Users — Identity, Access, and Per-User Privacy

## Status

**Proposed.** The settled decisions below are firm (operator + architect). The two
load-bearing forks (§ Open forks) go to a **decorrelated council** before this flips to
Accepted; the spec + board cards follow the council. Item 2 of the post-migration trio;
graduates `board/ideas/users-identity-privacy.md`.

> NB numbering: this is a **workspace** ADR (cross-cutting invariant, spans kernel +
> channel), distinct from the swarm-local sequence (whose 0016 is pg_search).

## Record Completeness

Draft — decision direction complete; the identity-ownership + conversation-enforcement
mechanisms are pending council.

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
6. **Admin may read others' conversations — NEVER by default; only via a separate,
   explicit, audited view, for support / diagnosing problems.** (Operator: Swarm is a
   work assistant, so support-reading is legitimate — but deliberate + audited, never
   ambient.)
7. **`groot` → role-based admin** (`owner`/`admin` role — already role-based in code); the
   concrete admin username is **hive-private config**, not hardcoded in `swarm/` + docs.
8. **Build the full model at once** (operator: surfaces the real advantages + problems),
   not phased.

## Open forks (→ decorrelated council before Accepted)

**A. Where identity lives — kernel vs channel vs hybrid.** Today auth is channel-side.
Tension: *microkernel stays small* vs *the kernel is the sole visibility authority* (if
conversations are kernel-owned + owner-enforced, the kernel must know the uuid identity).
→ **Architect lean (pending council):** **split** — the **uuid + user record** (login,
emails, scope/group grants) is **kernel-owned** (the authority for owner + scope);
**credential verification + session + SSO token exchange** stay **channel-side** (hive),
passing the authenticated `uuid + scopes` to the kernel; the person is **also projected as
a graph node** on the same uuid for facts-about-people (feeds item 3). **Do NOT** put
password hashes / SSO subjects into graph claim-edges.

**B. Where conversations live + how owner-only is enforced.** Today convlog is channel-side
(private volume).
→ **Architect lean (pending council):** make **owned-by-uuid a first-class kernel
visibility predicate** (enforce where every other visibility decision is made — no
split-brain); admin cross-user read = a separate, explicit, audited Core RPC. *Alternative:*
the channel keeps convlog with a strict owner check (leaner kernel, but splits the privacy
boundary across channel + kernel).

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

- **Channel-owned identity (status quo extended)** — leaner kernel, but splits the privacy
  boundary across channel + kernel (the split-brain fork B warns of). Council to weigh.
- **Person as a pure graph node (no auth table)** — clean "all users-as-data", but
  conflates security-sensitive auth with public-ish graph knowledge. Rejected for the auth
  layer; kept for the facts layer.
- **Phased rollout** — rejected by the operator (build all at once to surface real problems).
